---
title: "Perceptron Train XOR"
permalink: /writeups/perceptron-train-xor/
excerpt: "Writeup del reto 'Perceptron Train XOR' (AI Foundations I, categoria Artificial Intelligence, dificultad Easy, por LT 'syreal' Jones)."
date: 2026-09-11
categories:
  - AI Foundations I
  - AI
tags:
  - AI-Foundations-I
  - Easy
  - AI
---

Writeup del reto "Perceptron Train XOR" (AI Foundations I, categoria Artificial Intelligence, dificultad Easy, por LT 'syreal' Jones).

Reto web (no netcat): una app en el navegador entrena en vivo un perceptron de una sola capa sobre datos XOR, usando la regla de actualizacion clasica (solo los puntos mal clasificados generan actualizacion, sin weight decay). Como XOR no es linealmente separable, un unico perceptron no puede llegar al 100% de precision; el objetivo del reto es alcanzar el 75% de precision para demostrar que se entiende esa limitacion, lo cual revela la flag.

URL: http://aureolin-pixie.cylabacademy.net:62542/

Flag obtenida:  
academy{x0r_unl34rn4bl3_by_p3rc3ptr0ns_5307b2ae}

> **Flag:** `academy{x0r_unl34rn4bl3_by_p3rc3ptr0ns_5307b2ae}`

## 1. Reconocimiento del reto

El navegador con extension Claude-in-Chrome no estaba disponible en esta sesion ("Browser extension is not connected"), asi que en vez de interactuar con el slider desde una UI grafica se opto por leer el HTML/JS servidos y replicar la llamada de red directamente con curl.

Descarga de la pagina principal:

```text
curl -s http://aureolin-pixie.cylabacademy.net:62542/ | head -c 5000
```

La pagina describe el reto tal cual lo plantea el enunciado: "Tune the learning rate of a single-layer perceptron on XOR-style data. Each run performs a limited number of updates, and only misclassified points change the model. Since XOR is not linearly separable, 75% accuracy is considered a successful run." La UI expone: un slider/input numerico de learning rate, un boton "Run training", un canvas para visualizar la frontera de decision y una tabla de log de entrenamiento. Los scripts estaticos relevantes son "style.css" y "app.js".

## 2. Analisis del cliente (app.js) — como llama al backend

Se descarga la logica del cliente para entender que hace exactamente el boton "Run training":

```text
curl -s http://aureolin-pixie.cylabacademy.net:62542/app.js
```

Puntos clave del script:

- Al cargar la pagina, "loadConfig()" hace GET a "/config.json" para obtener el dataset, el numero maximo de pasos, los limites de learning rate y el modelo inicial.  
- Al pulsar "Run training", "runTraining()" hace:

```text
const res = await fetch("/train", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ learningRate: lr }),
});
const data = await res.json();
```

- La respuesta trae "history" (un registro paso a paso con pesos, sesgo y precision), "accuracy" final, "success" (booleano), un "message" y, si "success" es verdadero, el campo "flag" que la UI solo muestra si "data.success" es true.

Conclusion: todo el entrenamiento (el perceptron en si) corre en el servidor; el navegador solo envia el learning rate elegido y anima la respuesta. Por tanto, para obtener la flag basta con reproducir esa misma peticion POST directamente, sin necesidad de un navegador real.

## 3. Configuracion del dataset (config.json)

Se descarga la configuracion para conocer el dataset exacto y los parametros del entrenamiento:

```text
curl -s http://aureolin-pixie.cylabacademy.net:62542/config.json
```

Respuesta:

```text
{
  "points": [[-2, -2, 0], [2, 2, 0], [-2, 2, 1], [2, -2, 1]],
  "maxSteps": 16,
  "lrMin": 0.02,
  "lrMax": 20.0,
  "successThreshold": 0.75,
  "initialModel": {
    "step": 0, "sampleIndex": -1,
    "weights": [1.0, -1.0], "bias": 0.0, "accuracy": 0.25
  }
}
```

Es el XOR clasico en 4 puntos: (-2,-2)->0, (2,2)->0 (mismo signo en ambos ejes -> etiqueta 0) y (-2,2)->1, (2,-2)->1 (signos opuestos -> etiqueta 1). Con los pesos iniciales w=[1,-1], b=0, la activacion es w1*x + w2*y + b, prediccion 1 si es mayor o igual que 0. Verificacion manual de la precision inicial (0.25):

```text
(-2,-2): 1*-2 + -1*-2 + 0 =  0  -> predice 1, real 0 -> mal
( 2, 2): 1* 2 + -1* 2 + 0 =  0  -> predice 1, real 0 -> mal
(-2, 2): 1*-2 + -1* 2 + 0 = -4  -> predice 0, real 1 -> mal
( 2,-2): 1* 2 + -1*-2 + 0 =  4  -> predice 1, real 1 -> bien
```

1 acierto de 4 = 25%, coincide con el "accuracy": 0.25 del config. Umbral de exito: "successThreshold": 0.75, es decir, 3 de 4 puntos correctos.

## 4. Ejecucion del entrenamiento y obtencion de la flag

En lugar de mover el slider en el navegador, se reproduce la misma llamada que hace "app.js" directamente con curl, probando primero con learning rate = 1.0 (dentro del rango permitido 0.02-20.0):

```text
curl -s -X POST http://aureolin-pixie.cylabacademy.net:62542/train \
  -H "Content-Type: application/json" \
  -d '{"learningRate": 1.0}' | python3 -m json.tool
```

Respuesta (resumen):

```text
{
  "success": true,
  "history": [
    {
      "step": 1, "sampleIndex": 0, "point": [-2, -2, 0],
      "activation": 0.0, "prediction": 1, "error": -1,
      "weights": [3.0, 1.0], "bias": -1.0, "accuracy": 0.5
    },
    {
      "step": 2, "sampleIndex": 1, "point": [2, 2, 0],
      "activation": 7.0, "prediction": 1, "error": -1,
      "weights": [1.0, -1.0], "bias": -2.0, "accuracy": 0.75
    }
  ],
  "finalWeights": {"w1": 1.0, "w2": -1.0, "b": -2.0},
  "accuracy": 0.75,
  "message": "Nice work. You reached 75% accuracy, which is the best target for XOR with a single perceptron.",
  "flag": "academy{x0r_unl34rn4bl3_by_p3rc3ptr0ns_5307b2ae}",
  "successThreshold": 0.75
}
```

Con learning rate = 1.0 el perceptron convergio al objetivo en solo 2 actualizaciones (de un maximo de 16 pasos permitidos), alcanzando 75% de precision y devolviendo la flag directamente en el primer intento, sin necesidad de fuerza bruta sobre distintos valores de learning rate.

FLAG: academy{x0r_unl34rn4bl3_by_p3rc3ptr0ns_5307b2ae}

## 5. Fundamento: por que 75% es el maximo posible

El reto exige entender, no solo alcanzar, el limite del 75%. Los 4 puntos forman las esquinas de un cuadrado con el patron XOR: la etiqueta es 1 cuando "x" e "y" tienen signos opuestos, y 0 cuando tienen el mismo signo. Este patron es el ejemplo canonico de conjunto NO linealmente separable (no existe una unica linea recta -frontera de decision de un perceptron simple- que deje todos los "1" de un lado y todos los "0" del otro).

Sin embargo, dada la disposicion en cuadrado, cualquier linea recta bien elegida SI puede dejar 3 de los 4 puntos en el lado correcto (dejando siempre al menos 1 mal clasificado) -> maximo teorico de 3/4 = 75% de precision para un perceptron de una sola capa sobre este dataset. Por eso el propio enunciado fija el umbral de exito en 75% en lugar de 100%: es la cota superior alcanzable, no una meta arbitraria. La regla de actualizacion (ajustar pesos solo con los puntos mal clasificados, sin weight decay) hace que el modelo oscile perpetuamente tratando de corregir el punto que en cada momento queda mal clasificado, sin poder estabilizarse nunca en el 100%.
