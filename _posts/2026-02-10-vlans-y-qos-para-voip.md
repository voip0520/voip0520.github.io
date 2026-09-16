---
title: VLAN de voz y QoS para tráfico VoIP
excerpt: Cómo separar el tráfico de teléfonos IP y priorizarlo para eliminar cortes y jitter en la red LAN.
permalink: /proyectos/vlans-qos-voip/
date: 2026-02-10
categories:
- Redes
tags:
- vlan
- qos
- switching
- cisco
layout: writeup
techniques:
- vlan
- qos
- switching
- cisco
platform: Laboratorio autorizado
status: Completado
---

## Por qué separar la voz

Explica el problema (jitter, latencia, broadcast) que estás resolviendo.

## Configuración del switch

```
vlan 20
 name VOZ
!
interface range gi0/1 - 24
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 20
 mls qos trust cos
```

## Verificación

```
show vlan brief
show interfaces status
show mls qos interface gi0/1
```

## Resultado

Mediciones antes/después (latencia, jitter, MOS).
