---
title: Perceptron Train XNOR
permalink: /writeups/perceptron-train-xnor/
excerpt: Writeup del reto 'Perceptron Train XNOR' (AI Foundations I, categoria Artificial Intelligence, dificultad Easy, por LT 'syreal' Jones).
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

Writeup del reto "Perceptron Train XNOR" (AI Foundations I, categoria Artificial Intelligence, dificultad Easy, por LT 'syreal' Jones).

Mismo mecanismo que el reto hermano "Perceptron Train XOR": una app web entrena en vivo un perceptron de una sola capa, esta vez sobre datos XNOR, con la regla de actualizacion clasica (solo los puntos mal clasificados generan actualizacion, sin weight decay). Objetivo: alcanzar 75% de precision para revelar la flag.

URL: http://aureolin-pixie.cylabacademy.net:56515/

Flag obtenida:  
academy{xn0r_unl34rn4bl3_by_p3rc3ptr0ns_39fc9834}

Esta vez el archivo recoge los comandos exactos para reproducir el reto a mano, en orden.

> **Flag:** `academy{xn0r_unl34rn4bl3_by_p3rc3ptr0ns_39fc9834}`

## 1. Ver la configuracion del dataset

Comando:

```text
curl -s http://aureolin-pixie.cylabacademy.net:56515/config.json
```

Salida:

```text
{"points": [[-2, -2, 1], [2, 2, 1], [-2, 2, 0], [2, -2, 0]],
 "maxSteps": 16, "lrMin": 0.02, "lrMax": 20.0,
 "successThreshold": 0.75,
 "initialModel": {"step": 0, "sampleIndex": -1,
                   "weights": [1.0, 1.0], "bias": 0.0, "accuracy": 0.25}}
```

Es el mismo dataset en forma de cuadrado que el reto XOR pero con las etiquetas invertidas (patron XNOR: mismo signo en x/y -> 1, signos opuestos -> 0), y con pesos iniciales [1.0, 1.0] en vez de [1.0, -1.0]. El umbral de exito sigue siendo 75% (3 de 4 puntos), el maximo teorico para un perceptron simple sobre un dataset no separable linealmente.

## 2. Primer intento manual (learning rate = 1.0) -- no alcanza el umbral

Comando (mismo endpoint y formato que usa la app en app.js):

```text
curl -s -X POST http://aureolin-pixie.cylabacademy.net:56515/train \
  -H "Content-Type: application/json" \
  -d '{"learningRate": 1.0}' | python3 -m json.tool
```

Resultado: "success": false, "accuracy": 0.25, "flag": null.

Con lr=1.0 los 4 puntos se van corrigiendo uno a uno en un ciclo perfecto que se repite cada 4 pasos (accuracy 0.25 -> 0.25 -> 0.5 -> 0.25 -> 0.25 -> 0.25 -> 0.5 -> 0.25 ...), sin superar nunca el 50% en ningun paso del historial ni al final de los 16 pasos. Con este dataset y estos pesos iniciales, lr=1.0 no es suficiente (a diferencia del reto XOR, donde lr=1.0 si funcionaba).

## 3. Barrido manual de learning rates

Para no adivinar a ciegas, se recorre el rango permitido (lrMin=0.02, lrMax=20.0) probando varios valores y mirando solo el campo "success"/"accuracy" de cada respuesta:

```text
for lr in 0.02 0.05 0.1 0.15 0.2 0.25 0.3 0.4 0.5 0.6 0.7 0.75 0.8 0.9 \
          1.2 1.5 2 2.5 3 4 5 7 10 15 20; do
  res=$(curl -s -X POST http://aureolin-pixie.cylabacademy.net:56515/train \
    -H "Content-Type: application/json" -d "{\"learningRate\": $lr}")
  success=$(echo "$res" | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['success'], d['accuracy'])")
  echo "lr=$lr -> $success"
done
```

Resultado del barrido:

```text
lr=0.02 -> True 0.75      lr=0.6  -> False 0.5
lr=0.05 -> True 0.75      lr=0.7  -> False 0.25
lr=0.1  -> True 0.75      lr=0.75 -> False 0.25
lr=0.15 -> True 0.75      lr=0.8  -> False 0.25
lr=0.2  -> True 0.75      lr=0.9  -> False 0.5
lr=0.25 -> True 0.75      lr=1.2  -> False 0.25
lr=0.3  -> True 0.75      lr=1.5  -> False 0.25
lr=0.4  -> True 0.75      lr=2    -> False 0.25
lr=0.5  -> True 0.75      lr=2.5  -> False 0.25
                           lr=3    -> False 0.25
                           lr=4    -> True 0.75
                           lr=5    -> True 0.75
                           lr=7    -> True 0.75
                           lr=10   -> True 0.75
                           lr=15   -> True 0.75
                           lr=20   -> True 0.75
```

Patron observado: los learning rates pequenos (<=0.5) y los muy grandes (>=4) llegan al 75%; una franja intermedia (0.6-3) queda atrapada en ciclos por debajo del umbral. Cualquier valor de la zona que funciona sirve para el siguiente paso; se elige el minimo permitido, lr=0.02, por ser el mas facil de justificar (extremo del rango).

## 4. Comando final -- obtener la flag (learning rate = 0.02)

Comando:

```text
curl -s -X POST http://aureolin-pixie.cylabacademy.net:56515/train \
  -H "Content-Type: application/json" \
  -d '{"learningRate": 0.02}' | python3 -m json.tool
```

Respuesta completa:

```text
{
  "success": true,
  "history": [
    {"step": 1, "point": [-2,-2,1], "prediction": 0, "error": 1,
     "weights": [0.96, 0.96], "bias": 0.02, "accuracy": 0.25},
    {"step": 2, "point": [2,2,1],   "prediction": 1, "error": 0,
     "weights": [0.96, 0.96], "bias": 0.02, "accuracy": 0.25},
    {"step": 3, "point": [-2,2,0],  "prediction": 1, "error": -1,
     "weights": [1.0, 0.92],  "bias": 0.0,  "accuracy": 0.5},
    {"step": 4, "point": [2,-2,0],  "prediction": 1, "error": -1,
     "weights": [0.96, 0.96], "bias": -0.02, "accuracy": 0.75}
  ],
  "finalWeights": {"w1": 0.96, "w2": 0.96, "b": -0.02},
  "accuracy": 0.75,
  "message": "Nice work. You reached 75% accuracy, which is the best target for XNOR with a single perceptron.",
  "flag": "academy{xn0r_unl34rn4bl3_by_p3rc3ptr0ns_39fc9834}",
  "successThreshold": 0.75
}
```

Con lr=0.02 el modelo alcanza el 75% de precision en solo 4 actualizaciones (de un maximo de 16 permitidas).

FLAG: academy{xn0r_unl34rn4bl3_by_p3rc3ptr0ns_39fc9834}
