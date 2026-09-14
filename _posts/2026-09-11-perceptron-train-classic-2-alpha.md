---
title: "Perceptron Train Classic 2 Alpha"
permalink: /writeups/perceptron-train-classic-2-alpha/
excerpt: "Writeup del reto 'Perceptron Train Classic 2 Alpha' (AI Foundations I, categoria Artificial Intelligence, dificultad Easy, por LT 'syreal' Jones)."
date: 2026-09-11
categories:
  - AI Foundations I
  - AI
tags:
  - AI-Foundations-I
  - Easy
  - AI
---

Writeup del reto "Perceptron Train Classic 2 Alpha" (AI Foundations I, categoria Artificial Intelligence, dificultad Easy, por LT 'syreal' Jones).

Variante distinta a los retos hermanos XOR/XNOR/Hole in Middle: el dataset SI es linealmente separable, pero la flag no se revela con una unica ejecucion al 100% de precision. Hay que encontrar 5 learning rates distintos que cada uno alcance 100% de precision antes de que se revele la flag. El servidor lleva la cuenta de exitos de forma global (sin cookies de sesion).

URL: http://aureolin-pixie.cylabacademy.net:53012/

Flag obtenida:  
academy{perceptron_classic_2_alpha_5rates_bb5cb23b}

Este archivo recoge los comandos manuales para reproducir el reto, en orden.

> **Flag:** `academy{perceptron_classic_2_alpha_5rates_bb5cb23b}`

## 1. Ver la configuracion del dataset

Comando:

```text
curl -s http://aureolin-pixie.cylabacademy.net:53012/config.json | python3 -m json.tool
```

Salida:

```text
{
  "points": [
    [-2, 4, 1], [1, 1, 1], [-2, 0, 0], [3, -1, 1],
    [-3, -4, 0], [3, -4, 0], [4, 1, 1], [1, 3, 1],
    [-1, -3, 0], [-3, 2, 0], [2, -2, 0], [4, -2, 1]
  ],
  "maxSteps": 16,
  "lrMin": 0.02,
  "lrMax": 20.0,
  "successTarget": 5,
  "successCount": 0,
  "initialModel": {
    "step": 0, "sampleIndex": -1,
    "weights": [1.0, -1.0], "bias": 0.0, "accuracy": 0.5
  }
}
```

12 puntos, dataset linealmente separable (no es un patron XOR/XNOR/anillo). La clave del reto esta en los campos "successTarget": 5 y "successCount": 0: la app cuenta, en el servidor, cuantas ejecuciones distintas han llegado al 100% de precision, y solo revela la flag cuando esa cuenta llega a 5. Se confirmo con curl -i que el servidor NO establece cookie de sesion, es decir, el contador de exitos es un estado global compartido, no por sesion de usuario.

## 2. Primer intento manual (learning rate = 1.0)

Comando (mismo endpoint y formato que usa la app en app.js):

```text
curl -s -X POST http://aureolin-pixie.cylabacademy.net:53012/train \
  -H "Content-Type: application/json" \
  -d '{"learningRate": 1.0}' | python3 -m json.tool
```

Resultado relevante:

```text
"accuracy": 1.0,
"message": "Great run. This learning rate reached 100% accuracy. Find 4 more successful rates.",
"flag": null,
"successCount": 1,
"successTarget": 5
```

Con lr=1.0 el perceptron converge al 100% de precision en 11 de los 16 pasos permitidos. El mensaje de la API confirma explicitamente el mecanismo: cada tasa de aprendizaje que llega al 100% suma 1 al contador global, y hacen falta 5 para desbloquear la flag.

## 3. Intentos manuales adicionales -- completar el contador

Se prueban mas learning rates uno a uno, revisando el campo "successCount" de cada respuesta para saber cuantos exitos distintos llevamos acumulados. Comando repetido cambiando el valor de "learningRate":

```text
curl -s -X POST http://aureolin-pixie.cylabacademy.net:53012/train \
  -H "Content-Type: application/json" \
  -d '{"learningRate": <VALOR>}' \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print('accuracy=',d['accuracy'],'successCount=',d.get('successCount'),'flag=',d.get('flag'))"
```

Secuencia de intentos y resultado acumulado:

```text
lr=0.5  -> accuracy=0.8333  successCount=1  (no sube, no llega a 100%)
lr=2.0  -> accuracy=1.0     successCount=2  (segundo exito)
lr=5.0  -> accuracy=0.8333  successCount=2  (no sube)
lr=0.3  -> accuracy=0.9167  successCount=2  (no sube)
lr=0.02 -> accuracy=0.5833  successCount=2
lr=0.05 -> accuracy=0.75    successCount=2
lr=0.1  -> accuracy=0.9167  successCount=2
lr=0.15 -> accuracy=0.8333  successCount=2
lr=0.2  -> accuracy=0.9167  successCount=2
lr=0.25 -> accuracy=1.0     successCount=3  (tercer exito)
lr=0.4  -> accuracy=1.0     successCount=4  (cuarto exito)
```

Solo las tasas que llegan exactamente a "accuracy": 1.0 incrementan "successCount"; los intentos fallidos no cuentan ni penalizan, asi que se puede seguir probando sin limite hasta acertar. Con lr=0.4 el contador global ya esta en 4/5, a un solo exito de la flag.

## 4. Comando final -- quinto exito y obtencion de la flag (learning rate = 0.6)

Comando:

```text
curl -s -X POST http://aureolin-pixie.cylabacademy.net:53012/train \
  -H "Content-Type: application/json" \
  -d '{"learningRate": 0.6}' | python3 -m json.tool
```

Respuesta relevante:

```text
{
  "accuracy": 1.0,
  "successCount": 5,
  "successTarget": 5,
  "flag": "academy{perceptron_classic_2_alpha_5rates_bb5cb23b}",
  ...
}
```

Con lr=0.6 el perceptron vuelve a converger al 100% de precision, y al ser el quinto learning rate distinto que llega a ese resultado, el contador global alcanza 5/5 y la API devuelve la flag en la misma respuesta.

FLAG: academy{perceptron_classic_2_alpha_5rates_bb5cb23b}

Nota: como el contador es un estado global del servidor (sin cookies), en principio bastaria con que CUALQUIER cliente acumule 5 ejecuciones exitosas -- no hace falta que las 5 se ejecuten desde la misma sesion de curl ni en un orden particular, solo que sean 5 tasas de aprendizaje diferentes que cada una llegue a 100% de precision.
