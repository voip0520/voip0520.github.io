---
title: "Perceptron Train Hole in Middle"
excerpt: "Writeup del reto 'Perceptron Train Hole in Middle' (AI Foundations I, categoria Artificial Intelligence, dificultad Easy, por LT 'syreal' Jones)."
date: 2026-09-11
categories:
  - AI Foundations I
  - AI
tags:
  - AI-Foundations-I
  - Easy
  - AI
---

Writeup del reto "Perceptron Train Hole in Middle" (AI Foundations I, categoria Artificial Intelligence, dificultad Easy, por LT 'syreal' Jones).

Mismo mecanismo que los retos hermanos "Perceptron Train XOR" y "Perceptron Train XNOR": una app web entrena en vivo un perceptron de una sola capa (solo los puntos mal clasificados generan actualizacion, sin weight decay), esta vez sobre un patron "agujero en el medio": ocho puntos positivos rodeando un unico punto negativo en el centro. Objetivo: alcanzar 88.9% de precision para revelar la flag.

URL: http://aureolin-pixie.cylabacademy.net:53926/

Flag obtenida:  
academy{h0l3_1n_m1ddl3_unl34rn4bl3_l1n34rly_c85a6043}

Este archivo recoge unicamente los comandos manuales para reproducir el reto, en orden.

> **Flag:** `academy{h0l3_1n_m1ddl3_unl34rn4bl3_l1n34rly_c85a6043}`

## 1. Ver la configuracion del dataset

Comando:

```text
curl -s http://aureolin-pixie.cylabacademy.net:53926/config.json | python3 -m json.tool
```

Salida:

```text
{
  "points": [
    [0, 3, 1], [2, 2, 1], [3, 0, 1], [2, -2, 1],
    [0, -3, 1], [-2, -2, 1], [-3, 0, 1], [-2, 2, 1],
    [0, 0, 0]
  ],
  "maxSteps": 16,
  "lrMin": 0.02,
  "lrMax": 20.0,
  "successThreshold": 0.8888888888888888,
  "initialModel": {
    "step": 0, "sampleIndex": -1,
    "weights": [1.0, -1.0], "bias": 0.0,
    "accuracy": 0.5555555555555556
  }
}
```

Son 9 puntos: 8 positivos formando un anillo alrededor del origen y 1 negativo justo en el centro (0,0). El origen queda dentro del casco convexo de los 8 puntos positivos, asi que ninguna linea recta puede aislarlo como negativo sin dejar mal clasificado al menos un punto del anillo. Maximo teorico: 8 de 9 = 88.888...% -> coincide exactamente con "successThreshold". El umbral no es arbitrario: es la cota superior de cualquier clasificador lineal sobre este dataset.

## 2. Barrido manual de learning rates

En vez de adivinar, se recorre el rango permitido (lrMin=0.02, lrMax=20.0) probando varios valores y mirando solo "success"/"accuracy" de cada respuesta:

```text
for lr in 0.02 0.05 0.1 0.15 0.2 0.25 0.3 0.4 0.5 0.6 0.7 0.75 0.8 0.9 \
          1.0 1.2 1.5 2 2.5 3 4 5 7 10 15 20; do
  res=$(curl -s -X POST http://aureolin-pixie.cylabacademy.net:53926/train \
    -H "Content-Type: application/json" -d "{\"learningRate\": $lr}")
  success=$(echo "$res" | python3 -c "import json,sys; d=json.load(sys.stdin); print(d['success'], round(d['accuracy'],4))")
  echo "lr=$lr -> $success"
done
```

Resultado del barrido:

```text
lr=0.02 -> False 0.4444      lr=0.8  -> True  0.8889
lr=0.05 -> False 0.4444      lr=0.9  -> True  0.8889
lr=0.1  -> False 0.5556      lr=1.0  -> True  0.8889
lr=0.15 -> True  0.8889      lr=1.2  -> True  0.8889
lr=0.2  -> True  0.8889      lr=1.5  -> True  0.8889
lr=0.25 -> True  0.8889      lr=2    -> True  0.8889
lr=0.3  -> True  0.8889      lr=2.5  -> True  0.8889
lr=0.4  -> True  0.8889      lr=3    -> True  0.8889
lr=0.5  -> True  0.8889      lr=4    -> True  0.8889
lr=0.6  -> True  0.8889      lr=5    -> True  0.8889
lr=0.7  -> False 0.7778      lr=7    -> True  0.8889
lr=0.75 -> False 0.7778      lr=10   -> True  0.8889
                              lr=15   -> True  0.8889
                              lr=20   -> True  0.8889
```

Patron observado: casi todo el rango medio-alto (0.15-0.6 y 0.8 en adelante) alcanza el 88.9%; solo learning rates muy pequenos (<=0.1) y una franja estrecha (0.7-0.75) se quedan por debajo. Se elige lr=0.15 (el primer valor que funciona) para el paso final.

## 3. Comando final -- obtener la flag (learning rate = 0.15)

Comando:

```text
curl -s -X POST http://aureolin-pixie.cylabacademy.net:53926/train \
  -H "Content-Type: application/json" \
  -d '{"learningRate": 0.15}' | python3 -m json.tool
```

Respuesta (resumen, historial completo de 10 pasos hasta alcanzar el umbral):

```text
{
  "success": true,
  "history": [
    ... (pasos 1-9 corrigiendo puntos del anillo y luego el centro,
         precision fluctuando entre 0.4444 y 0.5556) ...
    {
      "step": 10, "point": [0, 3, 1], "prediction": 0, "error": 1,
      "weights": [-0.05, -0.10], "bias": 0.6,
      "accuracy": 0.8888888888888888
    }
  ],
  "finalWeights": {"w1": -0.04999999999999999, "w2": -0.10000000000000009, "b": 0.6},
  "accuracy": 0.8888888888888888,
  "message": "Nice work. You reached 88.9% accuracy, which is the best target for a single-line classifier on this ring-with-center dataset.",
  "flag": "academy{h0l3_1n_m1ddl3_unl34rn4bl3_l1n34rly_c85a6043}",
  "successThreshold": 0.8888888888888888
}
```

Con lr=0.15 el modelo alcanza el 88.9% de precision en el paso 10 (de un maximo de 16 permitidos), sacrificando siempre el punto central (0,0) -- el unico que ninguna recta puede clasificar bien a la vez que acierta los 8 puntos del anillo.

FLAG: academy{h0l3_1n_m1ddl3_unl34rn4bl3_l1n34rly_c85a6043}
