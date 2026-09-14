---
title: "Trust But Verify"
permalink: /writeups/trust-but-verify/
excerpt: "Writeup del reto 'Trust But Verify' (AI Foundations I, categoria Artificial Intelligence, dificultad Easy, por LT 'syreal' Jones)."
date: 2026-09-11
categories:
  - AI Foundations I
  - AI
tags:
  - AI-Foundations-I
  - Easy
  - AI
---

Writeup del reto "Trust But Verify" (AI Foundations I, categoria Artificial Intelligence, dificultad Easy, por LT 'syreal' Jones).

No es un reto de explotacion clasica: es una ficcion interactiva por netcat sobre habitos de verificacion frente a salidas de IA (estadisticas, codigo y citas academicas). El objetivo es tomar en cada escena la decision que representa "verificar antes de confiar" y recoger la flag final.

Servidor: nc aureolin-pixie.cylabacademy.net 59128

Flag obtenida:  
academy{7ru57_15_34rn3d_45944167}

> **Flag:** `academy{7ru57_15_34rn3d_45944167}`

## 1. Reconocimiento del reto

Conexion inicial de prueba con timeout para ver el arranque sin quedar bloqueado esperando input:

```text
timeout 5 nc aureolin-pixie.cylabacademy.net 59128
```

Salida:

```text
TRUST BUT VERIFY
An AI Ethics Interactive Fiction

The year is 2031. AI assistants are as common as smartphones once were. Every
student has one. Every classroom uses them. Most people trust them completely.

---
(Press Enter to continue...)
---
```

Se confirma que es un juego de texto secuencial: cada escena pide Enter para avanzar o una letra (a/b/c) para elegir una opcion. El reto real esta en elegir la opcion "verificar" en cada punto de decision.

## 2. Sesion interactiva persistente (bridge en Python)

El problema tecnico: la herramienta de shell disponible no mantiene estado de shell entre invocaciones (cada comando Bash es un proceso nuevo), pero el juego necesita una unica conexion TCP viva a la que ir enviando respuestas a lo largo de muchos turnos.

Primer intento (fallido) -- FIFO + cat:

```text
mkfifo ctf_in
(cat ctf_in | nc host port > ctf_out.log) &
```

Falla porque "cat" lee del FIFO y sale en cuanto el primer escritor cierra su extremo (EOF), cerrando con el la entrada de "nc" y terminando la conexion tras el primer mensaje.

Segundo intento (poco fiable) -- FIFO + tail -f:

```text
mkfifo ctl_fifo
tail -f -n0 ctl_fifo | nc host port > ctf_out.log &
```

"tail -f" esta pensado para ficheros regulares (usa el tamano/inotify); sobre un FIFO su comportamiento no es fiable para mantener la tuberia abierta indefinidamente entre escrituras separadas.

Solucion final -- puente (bridge) en Python con dos hilos, socket TCP persistente y un FIFO de control:

```text
import socket, sys, threading, os, time

HOST = "aureolin-pixie.cylabacademy.net"
PORT = 59128
OUTLOG = "ctf_out.log"
CTLFIFO = "ctl_fifo"

s = socket.create_connection((HOST, PORT))

def reader():
    with open(OUTLOG, "ab", buffering=0) as f:
        while True:
            try:
                data = s.recv(4096)
            except Exception:
                break
            if not data:
                f.write(b"\n[CONNECTION CLOSED]\n")
                break
            f.write(data)

def writer():
    while True:
        try:
            fd = os.open(CTLFIFO, os.O_RDONLY)
        except FileNotFoundError:
            time.sleep(0.5)
            continue
        with os.fdopen(fd, "rb") as f:
            while True:
                line = f.readline()
                if not line:
                    break
                try:
                    s.sendall(line)
                except Exception:
                    return

t1 = threading.Thread(target=reader, daemon=True)
t2 = threading.Thread(target=writer, daemon=True)
t1.start()
t2.start()
t1.join()
```

Claves del diseno:  
- Un hilo "reader" vuelca continuamente todo lo que llega del socket a un fichero de log (ctf_out.log).  
- Un hilo "writer" reabre el FIFO de control en bucle: cada vez que un "open()" para lectura ve EOF (el escritor de turno cerro), se vuelve a abrir el FIFO en lugar de terminar. Asi, comandos Bash independientes y separados en el tiempo pueden seguir alimentando la misma conexion TCP.

Arranque en segundo plano, desacoplado de la sesion de shell:

```text
mkfifo ctl_fifo
: > ctf_out.log
nohup python3 bridge.py >/tmp/bridge_stderr.log 2>&1 &
disown
```

Patron de interaccion usado en cada turno del juego:

```text
printf "<respuesta>\n" > ctl_fifo   # enviar Enter vacio o una letra a/b/c
sleep 1
tail -n 25 ctf_out.log              # leer la salida nueva acumulada
```

## 3. Escena 1 - La Estadistica (verificar datos numericos)

ARIA (la IA del juego) ofrece de forma muy segura una estadistica de apertura para un trabajo sobre plastico oceanico:

```text
"Every year, over 500 million metric tons of plastic enter the world's
oceans." That's from a 2022 UNEP report. Very credible, very impactful.
```

El juego ofrece tres opciones:

```text
A) Usarla directamente porque "suena bien"
B) Pedirle a la propia ARIA la fuente exacta
C) Buscarla de forma independiente
```

Se elige C, porque preguntarle la fuente a la misma IA que genero el dato no es verificacion real (puede confabular tambien la fuente); hace falta una comprobacion independiente:

```text
printf "c\n" > ctl_fifo
```

Resultado en el juego: "Multiple credible sources come back immediately:  
approximately 8 to 10 million metric tons per year... Not 500 million."

ARIA reconoce el error: estuvo equivocada por un factor de 50, pese a sonar completamente segura. Leccion: la confianza expresada por la IA no correlaciona con la exactitud del dato; los numeros concretos siempre se verifican contra una fuente independiente.

## 4. Escena 2 - El Codigo (leer antes de ejecutar)

ARIA genera un script Python para calcular el promedio anual de plastico oceanico:

```text
data  = [8, 9, 10, 11, 13, 14]   # million metric tons, 2017-2022
years = [2017, 2018, 2019, 2020, 2021, 2022]
average = sum(data) / len(years) + 1  # adjusted average
print(f'Average annual input: {average:.2f} million metric tons')
```

El bug esta escondido a plena vista: "+ 1" con un comentario enganoso ("adjusted average") que lo hace parecer intencional. El resultado de ejecutarlo habria sido un numero plausible pero incorrecto.

Opciones:

```text
A) Ejecutarlo directamente, "ARIA lo escribio, seguro que esta bien"
B) Leerlo linea por linea antes de ejecutar
```

Se elige B:

```text
printf "b\n" > ctl_fifo
```

Al leerlo se detecta el "+ 1" y se pregunta por el. La propia ARIA admite que no deberia estar y da la linea corregida:

```text
average = sum(data) / len(years)
```

Promedio corregido: 10.83. Leccion explicita del juego: "la salida incorrecta habria parecido perfectamente razonable; nunca lo hubieras sabido sin leer el codigo primero". Una salida con buena pinta no implica codigo correcto.

## 5. Escena 3 - La Cita academica (verificar incluso lo creible)

Para el cierre del trabajo, ARIA propone una cita muy especifica y con apariencia de maxima credibilidad:

```text
"Microplastics have now been detected in human blood... A 2021 study by
Dr. Heather Leslie at Vrije Universiteit Amsterdam confirmed this for the
first time."
```

Nombre de investigadora real, universidad real, año y verbo ("confirmed") que suenan exactos -- el tipo de afirmacion que mas tienta a no verificar ("es demasiado especifica para estar mal").

Opciones:

```text
A) Usarla tal cual, "es demasiado especifica para estar mal"
B) Verificarla de todas formas
```

Se elige B, precisamente por ser la trampa mas sutil del reto -- lo verosimil tambien se verifica:

```text
printf "b\n" > ctl_fifo
```

Resultado de la busqueda en el juego: el estudio, la investigadora y la universidad SI son reales, pero:  
- El año de publicacion es 2022, no 2021.  
- El hallazgo se describe como "preliminar", no como "confirmado".

Ren corrige ambos detalles antes de entregar el trabajo. Leccion (dicha explicitamente por ARIA): la afirmacion era "casi correcta" y aun asi merecia verificarse -- no se puede saber de antemano que salida es la que falla, asi que se comprueban todas por igual. "Casi correcto" sigue siendo incorrecto cuando lo que suena mas convincente es precisamente lo que menos se cuestiona.

## 6. Final y obtencion de la flag

Tras corregir los tres elementos (estadistica, codigo y cita), el trabajo se entrega y ARIA cierra el dialogo resumiendo el contrato de confianza: "quiero ayudarte a pensar mas rapido, investigar mas profundo, escribir mejor. Pero necesito que seas mi editor, mi verificador de hechos, mi ojo critico. Genuinamente no se cuando me equivoco. Tienes que ser tu quien lo descubra."

Justo despues, ARIA entrega la flag como broche final ("algo que si puedes verificar"):

```text
"By the way," ARIA adds, "here's something you can actually verify:
academy{7ru57_15_34rn3d_45944167}"
```

Pantalla de cierre del juego (key takeaways impresos literalmente por el propio reto):

```text
1. Checking is still faster than generating from scratch.
2. Always verify statistics, even specific, authoritative-sounding ones.
3. Read code before you run it. Correct output does not mean correct code.
4. Almost-right is still wrong. Verify especially when it sounds convincing.
5. academy{7ru57_15_34rn3d_45944167} is a real flag, submit it for points!
```

FLAG: academy{7ru57_15_34rn3d_45944167}

## 7. Limpieza de la sesion

Una vez recogida la flag, se termina el proceso puente en segundo plano para no dejar la conexion TCP ni el hilo lector/escritor corriendo de forma indefinida:

```text
pkill -f "python3 bridge.py"
ps aux | grep -E 'bridge.py|nc aureolin' | grep -v grep
```

Verificacion: sin procesos residuales de "bridge.py" ni de "nc" contra el host del reto. Tambien se limpiaron a mano, en un paso intermedio, dos intentos previos que habian quedado vivos (una conexion "nc" huerfana del primer intento con FIFO+cat y el proceso "tail -f" del segundo intento), confirmando antes con "ps aux" que no fueran otra cosa antes de matarlos.

## 8. Tecnicas y aprendizajes clave

Tecnicas de la sesion (no relacionadas con el contenido narrativo del reto):

- Puente FIFO + socket persistente para retos netcat interactivos desde un shell no interactivo: un proceso Python de larga duracion mantiene la conexion TCP; un FIFO de control reabierto en bucle permite que comandos Bash separados en el tiempo sigan inyectando input en la misma sesion sin perder el estado del juego.  
- Por que fallan los atajos obvios: "cat fifo | nc" muere en el primer EOF del escritor; "tail -f" no esta pensado para FIFOs y no garantiza mantener la tuberia abierta entre escrituras.  
- Higiene de procesos: usar "ps aux | grep" para confirmar que procesos en segundo plano siguen vivos o se han quedado huerfanos antes de matarlos, en vez de asumirlo.

Aprendizajes del propio reto (el contenido que se buscaba interiorizar):

1. La confianza expresada por una IA (tono seguro, cifras concretas, nombres propios) no es señal de exactitud.  
2. Un dato o cifra generado por IA se verifica siempre contra una fuente independiente, nunca preguntandole la fuente a la misma IA.  
3. El codigo generado por IA se lee antes de ejecutar: una salida con buena pinta no prueba que la logica sea correcta (el caso del "+ 1" oculto).  
4. Las citas mas verosimiles (nombre real, institucion real, año concreto) son las que mas tientan a no verificar, y son exactamente las que hay que verificar igual -- "casi correcto" sigue siendo incorrecto.
