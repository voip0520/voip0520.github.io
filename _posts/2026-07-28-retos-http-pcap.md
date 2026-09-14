---
title: "Retos HTTP con PCAP"
permalink: /writeups/retos-http-pcap/
excerpt: "WRITEUP - Resolución de retos HTTP con capturas PCAPObjetivo: analizar capturas de tráfico HTTP para extraer datos solicitados por el reto.Herramienta usada: Wireshark / tshark."
date: 2026-07-28
categories:
  - Máquinas
tags:
  - pcap
  - http
  - trafico
  - wireshark
---

> **Plataforma:** CTF

WRITEUP - Resolución de retos HTTP con capturas PCAPObjetivo: analizar capturas de tráfico HTTP para extraer datos solicitados por el reto.Herramienta usada: Wireshark / tshark.

## 1. Navegador usado

Archivo analizado: http1-2d3f819aa9773de1c698e32cb98b1b25.pcapFiltro usado en Wireshark:http.requestCampo revisado:User-AgentEvidencia encontrada:Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:93.0) Gecko/20100101 Firefox/93.0Respuesta / flag:Firefox/93.0

## 2. Idioma y país

Archivo analizado: http2-cc89e5d26367bdf5657cf466a45cd530.pcapFiltro usado:http.requestCampo revisado:Accept-LanguageEvidencia encontrada:Accept-Language: zh-CNInterpretación:zh = idioma chino; CN = China. El reto pide el país donde más se habla ese idioma.Respuesta / flag:China

## 3. Código HTTP de unknown.jpg

Archivo analizado: http3-0fcef3f43e4d753ce493c86df4232883.pcapFiltro usado en Wireshark:http contains "unknown.jpg"También se puede usar:http.request.uri contains "unknown.jpg"Luego se revisó la respuesta HTTP asociada a esa petición.Resultado encontrado:HTTP/1.1 404 Not FoundRespuesta / flag:404

## 4. Nombre de cookie

Archivo analizado: http4-21b32376f8e790aa30d08ea7b5ee1ba8.pcapFiltros usados:http.cookieo también:http contains "Cookie"Campo revisado:CookieEvidencia encontrada:Cookie: USER_TOKEN=...Nombre de la cookie:USER_TOKENRespuesta / flag:USER_TOKEN

## Comandos útiles con tshark

# Ver User-Agenttshark -r archivo.pcap -Y "http.request" -T fields -e http.user_agent# Ver idioma del navegadortshark -r archivo.pcap -Y "http.request" -T fields -e http.accept_language# Buscar recurso unknown.jpg y código de respuestatshark -r archivo.pcap -Y "http contains unknown.jpg"# Ver cookies HTTPtshark -r archivo.pcap -Y "http.cookie" -T fields -e http.cookie

## Resumen de flags

1. Navegador: Firefox/93.02. País por idioma: China3. Código HTTP unknown.jpg: 4044. Cookie usada: USER_TOKEN
