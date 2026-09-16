---
title: "Configurar una troncal SIP en Asterisk"
excerpt: "Registro de troncal, contextos y dialplan básico para salir a un proveedor VoIP sin dolores de cabeza."
permalink: /proyectos/asterisk-troncal-sip/
date: 2026-01-20
categories:
  - Telefonía IP
tags:
  - asterisk
  - sip
  - voip
---

## Escenario

Describe la central, la versión de Asterisk y el proveedor.

## Configuración de la troncal

```ini
[proveedor]
type=peer
host=sip.proveedor.com
username=USUARIO
secret=CLAVE
fromuser=USUARIO
context=entrantes
insecure=port,invite
disallow=all
allow=alaw,ulaw
```

## Dialplan

```ini
[salientes]
exten => _9X.,1,Dial(SIP/proveedor/${EXTEN:1},60)
 same => n,Hangup()
```

## Verificación

```bash
asterisk -rvvv
sip show registry
sip show peers
```

## Problemas comunes

- Audio en un solo sentido → NAT / `directmedia=no`.
- 403 Forbidden → credenciales o `fromdomain` mal configurado.
