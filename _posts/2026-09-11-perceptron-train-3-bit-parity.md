---
title: "Perceptron Train 3-Bit Parity"
permalink: /writeups/perceptron-train-3-bit-parity/
excerpt: "Writeup del reto 'Perceptron Train 3-Bit Parity' (AI Foundations I, categoria Artificial Intelligence, dificultad Easy, por LT 'syreal' Jones)."
date: 2026-09-11
categories:
  - AI Foundations I
  - AI
tags:
  - AI-Foundations-I
  - Easy
  - AI
---

Writeup del reto "Perceptron Train 3-Bit Parity" (AI Foundations I, categoria Artificial Intelligence, dificultad Easy, por LT 'syreal' Jones).

Extension a 3 dimensiones de la misma familia de retos que XOR/XNOR: una app web entrena en vivo un perceptron de una sola capa (solo los puntos mal clasificados generan actualizacion, sin weight decay) sobre datos de paridad de 3 bits, representados en 3D. Como la paridad no es linealmente separable por un unico plano, el objetivo es alcanzar 75% de precision para revelar la flag.

URL: http://aureolin-pixie.cylabacademy.net:62458/

Flag obtenida:  
academy{3b1t_p4r1ty_unl34rn4bl3_l1n34rly_5d1cd0f2}

Este archivo recoge las tecnicas y comandos exactos para reproducir el reto a mano, en orden.

> **Flag:** `academy{3b1t_p4r1ty_unl34rn4bl3_l1n34rly_5d1cd0f2}`

## 1. Ver la configuracion del dataset

Tecnica: antes de tocar el slider de la interfaz grafica, se descarga la configuracion que la propia app consulta al cargar (el mismo patron usado en los retos hermanos XOR/XNOR/Hole in Middle: la logica de entrenamiento vive en el servidor, no en el navegador).

Comando:

```text
curl -s http://aureolin-pixie.cylabacademy.net:62458/config.json | python3 -m json.tool
```

Salida:

```text
{
  "points": [
    [-2, -2, -2, 0], [-2, -2, 2, 1], [-2, 2, -2, 1], [-2, 2, 2, 0],
    [2, -2, -2, 1],  [2, -2, 2, 0],  [2, 2, -2, 0],  [2, 2, 2, 1]
  ],
  "maxSteps": 16,
  "lrMin": 0.02,
  "lrMax": 20.0,
  "successThreshold": 0.75,
  "dimensions": 3,
  "initialModel": {
    "step": 0, "sampleIndex": -1,
    "weights": [1.0, 1.0, 1.0], "bias": 0.0, "accuracy": 0.25
  }
}
```

Son los 8 vertices de un cubo (coordenadas +-2 en cada eje). La etiqueta es la paridad de 3 bits: 1 cuando el numero de coordenadas positivas es impar (1 o 3 positivas), 0 cuando es par (0 o 2 positivas) -- generalizacion directa del XOR de 2 bits a 3 dimensiones, y el ejemplo clasico de patron NO separable por un unico plano. El umbral de exito es 75% (6 de 8 puntos), el maximo teorico alcanzable con un solo plano separador sobre este dataset -- el propio nombre del campo "successThreshold" ya adelanta la cota.

## 2. Barrido manual de learning rates

Tecnica: en vez de mover el slider a ciegas, se reproduce la misma llamada POST /train que hace la interfaz (visible en app.js: fetch a "/train" con {"learningRate": lr} en el cuerpo), recorriendo el rango permitido (lrMin=0.02, lrMax=20.0) y mirando solo "success"/"accuracy" de cada respuesta:

```text
for lr in 0.02 0.05 0.1 0.15 0.2 0.25 0.3 0.4 0.5 0.6 0.7 0.75 0.8 0.9 \
          1.0 1.2 1.5 2 2.5 3 4 5 7 10 15 20; do
  res=$(curl -s -X POST http://aureolin-pixie.cylabacademy.net:62458/train \
    -H "Content-Type: application/json" -d "{\"learningRate\": $lr}")
  info=$(echo "$res" | python3 -c "import json,sys; d=json.load(sys.stdin); print(d.get('success'), round(d['accuracy'],4))")
  echo "lr=$lr -> $info"
done
```

Resultado del barrido:

```text
lr=0.02 -> False 0.25       lr=0.75 -> True 0.75
lr=0.05 -> False 0.25       lr=0.8  -> True 0.75
lr=0.1  -> False 0.25       lr=0.9  -> True 0.75
lr=0.15 -> True  0.75       lr=1.0  -> True 0.75
lr=0.2  -> True  0.75       lr=1.2  -> True 0.75
lr=0.25 -> True  0.75       lr=1.5  -> True 0.75
lr=0.3  -> True  0.75       lr=2    -> True 0.75
lr=0.4  -> True  0.75       lr=2.5  -> True 0.75
lr=0.5  -> True  0.75       lr=3    -> True 0.75
lr=0.6  -> True  0.75       lr=4    -> True 0.75
lr=0.7  -> True  0.75       lr=5, 7, 10, 15, 20 -> True 0.75
```

Patron observado: a diferencia de XOR/XNOR (donde habia una franja intermedia de learning rates que fallaba), aqui basta con superar un umbral bajo (lr >= 0.15) para que el perceptron alcance el 75% en cualquier punto del resto del rango permitido. Se elige lr=0.15 (el primer valor que funciona) para el paso final.

## 3. Comando final -- obtener la flag (learning rate = 0.15)

Comando:

```text
curl -s -X POST http://aureolin-pixie.cylabacademy.net:62458/train \
  -H "Content-Type: application/json" \
  -d '{"learningRate": 0.15}' | python3 -m json.tool
```

Respuesta (resumen; 11 pasos hasta alcanzar el umbral, de un maximo de 16):

```text
{
  "success": true,
  "history": [
    ... (pasos 1-10: la precision fluctua entre 0.25 y 0.5 mientras
         el modelo corrige distintos vertices del cubo) ...
    {
      "step": 11, "point": [-2, 2, -2, 1], "prediction": 0, "error": 1,
      "weights": [-0.2, 0.4, 0.4], "bias": 0.3,
      "accuracy": 0.75
    }
  ],
  "finalWeights": {"w1": -0.2, "w2": 0.4, "w3": 0.4, "b": 0.3},
  "accuracy": 0.75,
  "message": "Nice work. You reached 75% accuracy, the best target for 3-bit parity with a single perceptron.",
  "flag": "academy{3b1t_p4r1ty_unl34rn4bl3_l1n34rly_5d1cd0f2}",
  "successThreshold": 0.75
}
```

Con lr=0.15 el modelo alcanza el 75% de precision en el paso 11 (de un maximo de 16 permitidos): consigue clasificar bien 6 de los 8 vertices del cubo, sacrificando siempre los 2 restantes -- ningun plano puede separar la paridad completa de 3 bits.

FLAG: academy{3b1t_p4r1ty_unl34rn4bl3_l1n34rly_5d1cd0f2}
