---
title: "Navigator - Navigate CMS RCE y SUID php"
permalink: /writeups/navigator/
excerpt: "Enumeración de un servidor DNS/web Linux, bypass de autenticación en Navigate CMS mediante cookie, RCE, y escalada de privilegios a root abusando de un binario php con permiso SUID."
date: 2024-04-28
categories:
  - Máquinas
tags:
  - linux
  - navigate-cms
  - rce
  - dns
  - suid
  - metasploit
  - hacking
---

> **Máquina:** Navigator · **Plataforma:** Hacking · **SO:** Linux Debian 10 · **Dificultad:** Media

## 1. Reconocimiento

El descubrimiento de hosts sitúa el objetivo en `192.168.231.139`.

![Descubrimiento de host con arp-scan](/assets/informes/navigator/p03_01.jpg)

El escaneo de puertos con Nmap revela tres servicios expuestos: SSH (22), DNS (53) y HTTP (80).

![Escaneo de puertos con Nmap](/assets/informes/navigator/p03_02.jpg)

El escaneo de vulnerabilidades con scripts NSE y OpenVAS no arroja un vector directo, pero confirma un servidor web nginx sobre Debian.

![Escaneo de vulnerabilidades](/assets/informes/navigator/p04_03.jpg)

Al acceder al puerto 80 se muestra la página por defecto de nginx.

![Página por defecto de nginx](/assets/informes/navigator/p04_04.jpg)

## 2. Análisis y enumeración

La inspección del código fuente de la página revela una referencia a un dominio interno.

![Dominio en el código fuente](/assets/informes/navigator/p05_05.jpg)

Dado que el puerto 53 está abierto, se comprueba con `nslookup` que el servidor resuelve consultas DNS.

![Consulta DNS con nslookup](/assets/informes/navigator/p05_06.jpg)

![Verificación del servidor DNS](/assets/informes/navigator/p05_07.jpg)

Mediante `dnsrecon` y un reverse lookup se confirma la existencia del dominio `navigator.hm`, que se registra en `/etc/hosts`.

![Descubrimiento del dominio con dnsrecon](/assets/informes/navigator/p06_08.jpg)

![Registro del dominio en /etc/hosts](/assets/informes/navigator/p06_09.jpg)

Al navegar al dominio se expone información de la instalación (PHP 7.3, rutas de configuración).

![Información de la aplicación](/assets/informes/navigator/p06_10.jpg)

El fuzzing de directorios con `wfuzz` descubre el recurso `navigate`, que conduce a un panel de login de **Navigate CMS**.

![Panel de login de Navigate CMS](/assets/informes/navigator/p06_11.jpg)

Resumen del objetivo:

| Campo | Valor |
|---|---|
| IP | 192.168.231.139 |
| Sistema Operativo | Linux Debian 10 |
| Puertos | 22/SSH · 53/DNS · 80/HTTP |
| Vector | Navigate CMS RCE (bypass por cookie) |

## 3. Explotación

### Vía automatizada — Metasploit

Se localiza el módulo `exploit/multi/http/navigate_cms_rce`, correspondiente a la versión de Navigate CMS en ejecución.

![Módulo navigate_cms_rce en Metasploit](/assets/informes/navigator/p07_12.jpg)

Configurado el `RHOST` con el dominio, el módulo realiza un bypass de login y despliega el payload, obteniendo una sesión de Meterpreter como el usuario `www-data`.

![Sesión obtenida como www-data](/assets/informes/navigator/p08_13.jpg)

### Escalada de privilegios

Al carecer de privilegios, se transfiere `linpeas.sh` mediante un servidor HTTP temporal para enumerar vías de escalada.

![Servidor HTTP temporal](/assets/informes/navigator/p09_14.jpg)

![Descarga de linpeas en el objetivo](/assets/informes/navigator/p09_15.jpg)

![Permisos de ejecución a linpeas](/assets/informes/navigator/p09_16.jpg)

La auditoría revela credenciales en el histórico de MySQL (`.mysql_history`): un usuario y una contraseña reutilizados.

![Credenciales en .mysql_history](/assets/informes/navigator/p10_17.jpg)

Con `crackmapexec` se valida que la credencial del usuario `denisse` es válida por SSH.

![Validación de credenciales con crackmapexec](/assets/informes/navigator/p10_18.jpg)

Se establece sesión SSH como `denisse`.

![Acceso SSH como denisse](/assets/informes/navigator/p11_19.jpg)

linpeas había señalado un binario `php7.3` con permiso **SUID**. Se abusa de él para obtener una shell como root:

```bash
/usr/bin/php7.3 -r "pcntl_exec('/bin/sh', ['-p']);"
```

![Escalada a root vía SUID php](/assets/informes/navigator/p11_20.jpg)

### Vía manual — bypass por cookie

De forma complementaria, el bypass de autenticación se reproduce manualmente con Burp Suite, sustituyendo la cookie por la del exploit publicado en Exploit-DB (45561).

![Payload de bypass en el código del exploit](/assets/informes/navigator/p12_21.jpg)

![Interceptación de la petición de login con Burp](/assets/informes/navigator/p12_22.jpg)

Con la cookie manipulada se accede al panel del CMS.

![Acceso al panel de Navigate CMS](/assets/informes/navigator/p12_23.jpg)

Dentro del panel se crea una plantilla maliciosa apuntada a rutas del sistema, aprovechando la funcionalidad de plantillas para leer archivos sensibles.

![Creación de la plantilla maliciosa](/assets/informes/navigator/p13_24.jpg)

![Edición de la plantilla](/assets/informes/navigator/p13_25.jpg)

Se configura la plantilla para leer `/etc/passwd`.

![Plantilla apuntada a /etc/passwd](/assets/informes/navigator/p14_26.jpg)

![Lectura de /etc/passwd](/assets/informes/navigator/p14_27.jpg)

Se enumeran los usuarios del sistema y se localiza la primera bandera en el perfil de `denisse`.

![Lectura de la primera bandera](/assets/informes/navigator/p14_28.jpg)

## 4. Banderas

Con acceso como root se localizan las banderas mediante `find / -name bandera*.txt 2>/dev/null`.

![Búsqueda de banderas](/assets/informes/navigator/p15_29.jpg)

![Bandera del usuario denisse](/assets/informes/navigator/p15_30.jpg)

![Bandera de root](/assets/informes/navigator/p15_31.jpg)

| Usuario | Archivo | Bandera |
|---|---|---|
| denisse | bandera1.txt | 19019f428f02d94f958b9f709732a51e |
| root | bandera2.txt | e3b9c48f529685a5fca3e8a5d7d27e0a |

## 5. Herramientas utilizadas

Kali Linux, Nmap, OpenVAS, nslookup, dnsrecon, wfuzz, Metasploit, linpeas.sh, crackmapexec y Burp Suite.

## 6. Recomendaciones de mitigación

Actualizar Navigate CMS a una versión sin la vulnerabilidad de bypass, retirar el permiso SUID del binario php, evitar el almacenamiento de credenciales en histories de shell/MySQL y aplicar contraseñas robustas y no reutilizadas entre servicios.
