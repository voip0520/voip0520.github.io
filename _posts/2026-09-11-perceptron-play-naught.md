---
title: Perceptron Play Naught
permalink: /writeups/perceptron-play-naught/
excerpt: Writeup del reto 'Perceptron Play Naught' (AI Foundations I, categoria Artificial Intelligence, dificultad Easy, por LT 'syreal' Jones).
date: 2026-09-11
categories:
- AI Foundations I
- AI
tags:
- AI-Foundations-I
- Easy
- AI
layout: writeup
techniques:
- AI Foundations I
- Easy
- AI
platform: Laboratorio autorizado
status: Completado
---

Writeup del reto "Perceptron Play Naught" (AI Foundations I, categoria Artificial Intelligence, dificultad Easy, por LT 'syreal' Jones).

A diferencia de los retos "Perceptron Train *" (donde el servidor ejecuta la regla de aprendizaje del perceptron por nosotros), aqui el ajuste de pesos y bias es completamente MANUAL: es un playground interactivo por netcat con comandos SET/ADJUST/CHECK. Los mismos valores de x del "1D Charlie puzzle" (no separables en 1 sola dimension) se elevan a 2D anadiendo un valor de y, de forma que SI se pueden separar con una sola linea. El objetivo es encontrar a mano unos pesos w1, w2 y un bias b que separen perfectamente los puntos etiquetados.

Servidor: nc aureolin-pixie.cylabacademy.net 65220

Flag obtenida:  
academy{n4ught_bu7_53p4r4b13_9f26db48}

Este archivo recoge las tecnicas y comandos para reproducir el reto a mano, en orden.

> **Flag:** `academy{n4ught_bu7_53p4r4b13_9f26db48}`

## 1. Reconocimiento del reto y lectura de los puntos

Conexion de prueba con timeout para ver el menu inicial sin quedar bloqueado esperando input:

```text
timeout 5 nc aureolin-pixie.cylabacademy.net 65220
```

El servidor imprime de entrada el menu de comandos disponibles y, sin pedirlo, ya muestra el grafico ASCII y la tabla de puntos con los pesos iniciales (w1=1, w2=-1, b=0):

```text
Commands:
  SHOW      redraw the ASCII graph and point table
  SET w1 w2 b     set the weights/bias directly (floats are fine)
  ADJUST dw1 dw2 db   add offsets to the current weights
  POINTS    list the training points with their target labels
  CHECK     verify every point; prints the flag when perfect
  RESET     go back to the starting weights (w1=1.0, w2=-1.0, b=0.0)
  HELP / EXIT / QUIT

point    label  perceptron  activation
------   -----  ----------  ----------
(-4,-1)     0        0        -3
(-1,+2)     1        0        -3
(+0,-1)     0        1        1
(+0,+2)     1        0        -2
(+2,-1)     0        1        3
(+3,+1)     1        1        2
(+4,+2)     1        1        2
```

Con los pesos por defecto, 3 de los 7 puntos ya salen mal clasificados (columna "perceptron" distinta de "label", marcados con "x" en el grafico ASCII). La leyenda del propio juego explica el criterio: activacion = w1*x + w2*y + b, prediccion 1 si es mayor o igual que 0.

## 2. Analisis manual -- elegir los pesos a mano

Tecnica: en vez de tantear con ADJUST a ciegas, se listan los 7 puntos con POINTS/lo ya mostrado y se buscan visualmente ejes o combinaciones donde las dos clases queden separadas de forma trivial.

Puntos (x, y, label):

```text
(-4, -1, 0)
(-1,  2, 1)
( 0, -1, 0)
( 0,  2, 1)
( 2, -1, 0)
( 3,  1, 1)
( 4,  2, 1)
```

Se observa que TODOS los puntos de clase 0 tienen y = -1 (negativo) y TODOS los de clase 1 tienen y >= 1 (positivo); la coordenada x no aporta ninguna informacion util para separar las clases (de hecho, en la version 1D del puzzle -- solo con x -- las clases se solapan y no son separables, tal como indica el enunciado del reto). Por tanto basta con un clasificador que dependa unicamente de "y":

```text
activacion = w1*x + w2*y + b   con   w1 = 0, w2 = 1, b = 0
           = y
```

Verificacion manual antes de aplicarlo:

```text
(-4,-1,0): y=-1 < 0 -> predice 0  (correcto)
(-1, 2,1): y= 2 >=0 -> predice 1  (correcto)
( 0,-1,0): y=-1 < 0 -> predice 0  (correcto)
( 0, 2,1): y= 2 >=0 -> predice 1  (correcto)
( 2,-1,0): y=-1 < 0 -> predice 0  (correcto)
( 3, 1,1): y= 1 >=0 -> predice 1  (correcto)
( 4, 2,1): y= 2 >=0 -> predice 1  (correcto)
```

Los 7 puntos quedan bien clasificados con w1=0, w2=1, b=0 -- una linea horizontal (y=0) que separa las dos clases.

## 3. Sesion interactiva persistente (mismo patron bridge que otros retos netcat)

Igual que en el reto "Trust But Verify", se necesita mantener una unica conexion TCP viva a lo largo de varios comandos, pero la herramienta de shell disponible no conserva estado entre invocaciones separadas. Se reutiliza el mismo patron de puente (bridge) en Python: un socket persistente, un hilo que vuelca todo lo recibido a un log, y un hilo que reabre en bucle un FIFO de control para poder inyectar comandos desde llamadas Bash independientes.

Arranque:

```text
mkfifo naught_ctl_fifo
: > naught_out.log
nohup python3 bridge2.py >/tmp/bridge2_stderr.log 2>&1 &
disown
```

(El script "bridge2.py" es identico en estructura al usado en "Trust But Verify": conecta a HOST/PORT indicados, un hilo "reader" hace socket.recv() en bucle y escribe a "naught_out.log", un hilo "writer" reabre "naught_ctl_fifo" cada vez que ve EOF y reenvia cada linea leida al socket con sendall().)

Patron de interaccion en cada turno:

```text
printf "<COMANDO>\n" > naught_ctl_fifo
sleep 1
tail -n 25 naught_out.log
```

## 4. Comandos ejecutados y obtencion de la flag

Paso 1 -- fijar los pesos calculados a mano:

```text
printf "SET 0 1 0\n" > naught_ctl_fifo
```

Salida tras el comando (grafico ASCII y tabla, todos los puntos ya correctos):

```text
+4         |
+3         |
+2       1 1       1
+1         |     1
+0 / / / / / / / / /
-1 0       0   0
-2         |
...
Current weights -> w1: 0, w2: 1, b: 0

  point    label  perceptron  activation
  ------   -----  ----------  ----------
  (-4,-1)     0        0        -1
  (-1,+2)     1        1        2
  (+0,-1)     0        0        -1
  (+0,+2)     1        1        2
  (+2,-1)     0        0        -1
  (+3,+1)     1        1        1
  (+4,+2)     1        1        2
```

Ya no queda ningun punto marcado con "x": los 7 puntos coinciden entre "label" y "perceptron".

Paso 2 -- verificar y obtener la flag:

```text
printf "CHECK\n" > naught_ctl_fifo
```

Salida:

```text
Perfect! All points are classified correctly.
academy{n4ught_bu7_53p4r4b13_9f26db48}

[CONNECTION CLOSED]
```

El propio servidor cierra la conexion tras entregar la flag. Limpieza final del proceso puente:

```text
pkill -f "python3 bridge2.py"
```

FLAG: academy{n4ught_bu7_53p4r4b13_9f26db48}
