---
title: Perceptron Train Classic 0
permalink: /writeups/perceptron-train-classic-0/
excerpt: Writeup del reto 'Perceptron Train Classic 0' (AI Foundations I, categoria Artificial Intelligence, dificultad Easy, por LT 'syreal' Jones).
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

Writeup del reto "Perceptron Train Classic 0" (AI Foundations I, categoria Artificial Intelligence, dificultad Easy, por LT 'syreal' Jones).

El mas sencillo de la serie de retos de perceptron: mismo dataset separable que "Perceptron Train Classic 1" (dos clusters en diagonales opuestas), pero aqui NO hay contador de exitos acumulados ("successTarget"/"successCount") como en "Classic 1" o "Classic 2 Alpha" -- basta con UNA sola ejecucion que alcance 100% de precision para que la API devuelva la flag directamente en esa misma respuesta.

URL: http://aureolin-pixie.cylabacademy.net:63867/

Flag obtenida:  
academy{perceptron_classic_mode_583cf994}

Este archivo recoge los comandos manuales para reproducir el reto, en orden.

> **Flag:** `academy{perceptron_classic_mode_583cf994}`

## 1. Ver la configuracion del dataset

Comando:

```text
curl -s http://aureolin-pixie.cylabacademy.net:63867/config.json | python3 -m json.tool
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
  "lrMax": 2.0,
  "initialModel": {
    "step": 0, "sampleIndex": -1,
    "weights": [1.0, -1.0], "bias": 0.0, "accuracy": 0.5
  }
}
```

Mismos 8 puntos que "Classic 1" (cuadrante inferior-izquierdo con etiqueta 0, cuadrante superior-derecho con etiqueta 1; separacion muy holgada). Dos diferencias notables respecto a las variantes anteriores: el rango de learning rate permitido es mas estrecho (lrMax=2.0 en vez de 20.0) y no aparecen los campos "successTarget"/"successCount" -- indicio de que aqui no hace falta acumular varios exitos, con uno solo basta.

## 2. Comando de entrenamiento y obtencion de la flag (learning rate = 1.0)

Comando:

```text
curl -s -X POST http://aureolin-pixie.cylabacademy.net:63867/train \
  -H "Content-Type: application/json" \
  -d '{"learningRate": 1.0}' | python3 -m json.tool
```

Respuesta (resumen; primeros pasos hasta la convergencia):

```text
{
  "success": true,
  "message": "Perfect! You found a learning rate that reaches 100% accuracy in 16 steps.",
  "history": [
    {"step": 1, "point": [-4,-2,0], "prediction": 0, "error": 0,
     "weights": [1.0, -1.0], "bias": 0.0,  "accuracy": 0.5},
    {"step": 2, "point": [-3,-4,0], "prediction": 1, "error": -1,
     "weights": [4.0, 3.0],  "bias": -1.0, "accuracy": 1.0},
    ... (pasos 3 a 16: todos los puntos ya se clasifican bien,
         error=0 en cada uno, accuracy se mantiene en 1.0) ...
  ],
  "finalWeights": {"w1": 4.0, "w2": 3.0, "b": -1.0},
  "accuracy": 1.0,
  "flag": "academy{perceptron_classic_mode_583cf994}"
}
```

Con lr=1.0 el modelo corrige el unico punto mal clasificado en el paso 2 y se mantiene al 100% de precision durante el resto de los 16 pasos. Al no existir aqui un contador de exitos acumulados, la flag aparece en la misma respuesta de la primera ejecucion exitosa, sin necesidad de repetir la llamada con otros learning rates.

FLAG: academy{perceptron_classic_mode_583cf994}
