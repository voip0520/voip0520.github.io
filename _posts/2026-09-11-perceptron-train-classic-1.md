---
title: "Perceptron Train Classic 1"
excerpt: "Writeup del reto 'Perceptron Train Classic 1' (AI Foundations I, categoria Artificial Intelligence, dificultad Easy, por LT 'syreal' Jones)."
date: 2026-09-11
categories:
  - AI Foundations I
  - AI
tags:
  - AI-Foundations-I
  - Easy
  - AI
---

Writeup del reto "Perceptron Train Classic 1" (AI Foundations I, categoria Artificial Intelligence, dificultad Easy, por LT 'syreal' Jones).

Misma mecanica que "Perceptron Train Classic 2 Alpha": hay que encontrar 5 learning rates distintos que cada uno alcance 100% de precision antes de que la API revele la flag (contador global "successCount"/"successTarget", sin cookies de sesion). La diferencia es el dataset: aqui los puntos forman dos grupos claramente separados en diagonales opuestas, mucho mas facil de separar linealmente que el de "Classic 2 Alpha".

URL: http://aureolin-pixie.cylabacademy.net:55574/

Flag obtenida:  
academy{perceptron_classic_5rates_432f0bf5}

Este archivo recoge los comandos manuales para reproducir el reto, en orden.

> **Flag:** `academy{perceptron_classic_5rates_432f0bf5}`

## 1. Ver la configuracion del dataset

Comando:

```text
curl -s http://aureolin-pixie.cylabacademy.net:55574/config.json | python3 -m json.tool
```

Salida:

```text
{
  "points": [
    [-4, -2, 0], [-3, -4, 0], [-2, -3, 0], [-3, -1, 0],
    [3, 4, 1], [4, 2, 1], [2, 3, 1], [3, 1, 1]
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

8 puntos en dos grupos bien separados: 4 con etiqueta 0 en el cuadrante inferior-izquierdo (todas las coordenadas negativas) y 4 con etiqueta 1 en el cuadrante superior-derecho (todas positivas). A diferencia de "Classic 2 Alpha" la separacion es muy holgada, asi que se espera que la mayoria de learning rates converjan sin problema. Igual que en esa variante, "successTarget": 5 y "successCount": 0 indican que hacen falta 5 ejecuciones distintas al 100% de precision para desbloquear la flag.

## 2. Comandos manuales -- acumular 5 tasas de aprendizaje exitosas

Comando reutilizable, cambiando el valor de "learningRate" en cada intento:

```text
curl -s -X POST http://aureolin-pixie.cylabacademy.net:55574/train \
  -H "Content-Type: application/json" \
  -d '{"learningRate": <VALOR>}' \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['accuracy'], d.get('successCount'), d.get('flag'))"
```

Secuencia de intentos ejecutada (dado lo separable del dataset, las 5 primeras tasas probadas funcionaron, sin ningun fallo):

```text
lr=1.0  -> accuracy=1.0  successCount=1  flag=None
lr=2.0  -> accuracy=1.0  successCount=2  flag=None
lr=0.5  -> accuracy=1.0  successCount=3  flag=None
lr=0.25 -> accuracy=1.0  successCount=4  flag=None
lr=0.4  -> accuracy=1.0  successCount=5  flag=academy{perceptron_classic_5rates_432f0bf5}
```

Al llegar el contador global a 5/5 con el quinto intento (lr=0.4), la misma respuesta ya trae la flag en el campo "flag", sin necesidad de una llamada adicional.

## 3. Comando final -- respuesta completa con la flag

Ultimo comando ejecutado (el que completa el contador):

```text
curl -s -X POST http://aureolin-pixie.cylabacademy.net:55574/train \
  -H "Content-Type: application/json" \
  -d '{"learningRate": 0.4}' | python3 -m json.tool
```

Campos relevantes de la respuesta:

```text
{
  "accuracy": 1.0,
  "successCount": 5,
  "successTarget": 5,
  "flag": "academy{perceptron_classic_5rates_432f0bf5}",
  ...
}
```

FLAG: academy{perceptron_classic_5rates_432f0bf5}

Nota: como en "Classic 2 Alpha", el contador es un estado global del servidor (sin cookies de sesion): basta con acumular 5 ejecuciones exitosas con learning rates distintos, sin importar el orden ni que provengan de la misma sesion de curl.
