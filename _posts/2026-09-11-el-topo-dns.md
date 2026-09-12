---
title: "El Topo DNS"
excerpt: "Writeup DFIR de la maquina «El Topo DNS» (TheHackersLabs, categoria Seguridad Defensiva / Linux), objetivo 192.168.143.140."
date: 2026-09-11
categories:
  - TheHackersLabs
  - DFIR
tags:
  - TheHackersLabs
  - DFIR
---

Writeup DFIR de la maquina «El Topo DNS» (TheHackersLabs, categoria Seguridad Defensiva / Linux), objetivo 192.168.143.140.

La landing page del objetivo es un senuelo (troll) y no contiene ningun log. La evidencia real esta dentro del propio disco de la VM descargable, en /home/auditor/dfir_eltopo/, y se llego a ella descargando la VM (.ova), extrayendo su disco con 7z y leyendolo con debugfs, sin montar el disco ni usar privilegios de root.

Respuestas obtenidas:

1. IP externa que sirvio el stager p.sh       -> 162.248.1.100  
2. Fichero PHP, punto de entrada              -> upload.php  
3. FQDN del primer beaconing C2               -> 1.beacon.c2.eltopo.thl  
4. Dominio raiz de exfiltracion del shadow    -> eltopo.thl  
5. Protocolo de pivoting a 10.0.0.50          -> FTP  
6. Usuario del movimiento lateral             -> devuser  
7. Contrasena del movimiento lateral          -> developer123  
8. Fichero exacto exfiltrado                  -> client_database_backup.zip

## 1. Reconocimiento inicial

Antes de tocar logs hay que confirmar que expone realmente el objetivo. Escaneo completo de los 65535 puertos TCP:

```text
nmap -Pn -p- --min-rate 3000 -T4 192.168.143.140
```

Resultado:

```text
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
```

Solo tres puertos, confirmado tambien por un escaneo UDP de los 200 puertos mas comunes (sin nada relevante) y repitiendo el TCP mas tarde en la sesion (mismo resultado). Verificacion de version de servicios:

```text
nmap -Pn -sV -p21,22,80 --version-intensity 9 192.168.143.140

21/tcp open  ftp     vsftpd 2.0.8 or later
22/tcp open  ssh     OpenSSH 8.4p1 Debian 5+deb11u5
80/tcp open  http    Apache httpd 2.4.65 (Debian)
```

Nota importante: el enunciado con las 8 preguntas y los botones "Comprobar" NO vive en 192.168.143.140 -- vive en la plataforma (labs.thehackerslabs.com/machine/146, requiere sesion). El .140 es solo la IP de la maquina a investigar. Confundir ambas cosas costo varias vueltas en falso: pedirle a un fetch sin sesion esa URL solo devuelve la pantalla de login.

## 2. Callejones sin salida (senuelos)

Antes de llegar a la evidencia real se agotaron varias vias que parecian prometedoras. Se documentan porque descartarlas con evidencia (no por intuicion) es parte del analisis.

a) Imagen zapp.jpg -- La landing page sirve una imagen "hacker" con un div oculto (display:none) con base64 anidado 4 veces:

```text
curl -s -o zapp.jpg http://192.168.143.140/zapp.jpg
exiftool -a -u -g1 -ee zapp.jpg
```

Decodificar el base64 oculto 4 veces seguidas da "cuatrocuatroveces" -- un chiste autorreferencial sobre el "4444" que ya aparece en texto plano al lado. Probar steghide/stegseek contra la imagen con el diccionario completo rockyou.txt (14M passwords):

```text
stegseek zapp.jpg rockyou.txt -xf out.txt
[!] error: Could not find a valid passphrase.
```

Sin esteganografia real. Al mirar la imagen se resuelve solo: es un logo que dice literalmente "TROLL -- the funny coffee".

b) FTP anonimo -- login.txt y secret.txt (accesibles por FTP anonimo) contienen un acertijo en leetspeak ("Ojo con el cafe bien preparado, a veces la pista esta en la taza") y el nombre del creador de la maquina (puerto4444) trozeado en lineas sueltas. Ningun dato de ahi resulto ser una credencial valida.

c) Fuerza bruta SSH / FTP y directorios -- Diccionario dirigido (15 usuarios x 27 contrasenas, construido con todas las pistas del reto) contra SSH y FTP via hydra:

```text
hydra -L users.txt -P passes.txt -t 4 -f ssh://192.168.143.140
hydra -L users.txt -P passes.txt -t 4 -f ftp://192.168.143.140
```

Resultado: 0 credenciales validas en ambos servicios. Bruteforce de directorios web con feroxbuster (~30000 rutas x 8 extensiones):

```text
feroxbuster -u http://192.168.143.140/ -w raft-medium-directories.txt -t 100 -x php,txt,zip,log,pcap,json,csv,html
```

Sin backups, sin paneles, sin logs expuestos por HTTP. El backdoor conocido de vsftpd 2.3.4 (usuario:) -> puerto 6200) tampoco aplicaba (conexion rechazada).

Conclusion de esta fase: el objetivo en vivo no expone la evidencia por red. Hubo que conseguirla de otra forma: descargando la maquina completa desde la plataforma.

## 3. Descargar y extraer la VM (.ova)

La pagina de la maquina en la plataforma (labs.thehackerslabs.com/machine/146) tiene un boton "Descargar" que entrega la VM completa como .zip con un .ova dentro (formato VirtualBox). URL directa de descarga proporcionada por el usuario:

```text
https://thehackerslabs.com/maquinas/The%20Hackers%20Labs%20-%20El%20Topo%20DNS.zip
```

Descarga y extraccion:

```text
curl -s -L -o "El Topo DNS.zip" "https://thehackerslabs.com/maquinas/..."
unzip "El Topo DNS.zip"
```

Un .ova es, en el fondo, un tar con el descriptor OVF y el disco virtual:

```text
tar -tvf "The Hackers Labs - El Topo DNS.ova"

The Hackers Labs - El Topo DNS.ovf
The Hackers Labs - El Topo DNS-disk001.vmdk   (1.3 GB)
The Hackers Labs - El Topo DNS.mf
```

Extraer solo el .vmdk del tar:

```text
tar -xvf "The Hackers Labs - El Topo DNS.ova" "The Hackers Labs - El Topo DNS-disk001.vmdk"
```

El .vmdk viene en formato streamOptimized (comprimido con zlib), por lo que no se puede loopmount directamente. 7-Zip si entiende ese formato y expande cada particion del disco a un archivo .img independiente:

```text
7z l "The Hackers Labs - El Topo DNS-disk001.vmdk"
-> Method = "streamOptimized" zlib Marker

7z x "The Hackers Labs - El Topo DNS-disk001.vmdk" -o./extract_disk
```

Resultado: 0.img (20 GB, particion raiz), 1.img (976 MB, swap), 2 (1 MB, tabla de particiones). Identificar los filesystems:

```text
file extract_disk/0.img
-> Linux rev 1.0 ext4 filesystem data (needs journal recovery)
file extract_disk/1.img
-> Linux swap file
```

Aviso practico: la extraccion NO es sparse -- escribe los 20 GB completos. Conviene borrar el .zip y el .ova en cuanto se tiene el .vmdk, y borrar el .vmdk en cuanto se tiene el .img, para no llenar el disco.

## 4. Leer el disco sin montar (debugfs)

Montar un .img con "mount -o loop" pide privilegios de root, y no habia sudo disponible en esta maquina de trabajo. Alternativa: debugfs (parte de e2fsprogs) lee un filesystem ext4 directamente desde el archivo, SIN montarlo y SIN privilegios -- ideal para forense de solo lectura.

Explorar el arbol de directorios:

```text
debugfs -R "ls -l /" 0.img
debugfs -R "ls -l /home" 0.img
-> ... auditor   (uid 1001)

debugfs -R "ls -l /home/auditor" 0.img
-> ... dfir_eltopo

debugfs -R "ls -l /home/auditor/dfir_eltopo" 0.img

652821  etc_shadow_artifact.txt   189
652822  ftp.log                   638
652823  access.log                405097
652824  dns.log                   78000
```

Cuatro ficheros, uno por cada pista del enunciado. Volcar la carpeta completa a disco local con un solo comando:

```text
debugfs -R "rdump /home/auditor/dfir_eltopo dfir_out" 0.img
```

(rdump imprime avisos de "Operation not permitted while changing ownership" porque no se es root, pero los ficheros SI se copian correctamente con el contenido intacto -- el aviso es solo por el intento fallido de conservar el propietario original.)

## 5. access.log -- punto de entrada y stager

access.log tiene 5000 lineas, y el 99% es ruido sintetico (peticiones benignas desde 10.1.1.x a /index.html, /contact.php, /images/logo.png, etc.) pensado para obligar a filtrar en vez de leer el archivo entero. Filtrando por las rutas mencionadas en el enunciado (upload.php, p.sh) aparecen exactamente dos lineas con IPs relevantes:

```text
grep -i "p\.sh\|upload\.php" access.log

1.2.3.4 - - [10/nov/2025:09:10:13 +0100] "POST /upload.php HTTP/1.1" 200 150
192.168.1.10 - - [10/nov/2025:09:10:13 +0100] "GET http://162.248.1.100/p.sh HTTP/1.1" 200 1024
```

Lectura de las dos lineas:

- La primera es la explotacion: 1.2.3.4 (IP externa del atacante) hace POST a upload.php -- responde la pregunta 2 (punto de entrada = upload.php).  
- La segunda es el propio servidor ya comprometido (192.168.1.10) descargando el stager desde 162.248.1.100 -- esa IP responde la pregunta 1 (quien sirvio p.sh).

## 6. dns.log -- beaconing y exfiltracion

1008 lineas, mismo patron de ruido (www.google.com, www.updates.ubuntu.com, www.cdn.example.com...) mezclado con dos familias de subdominios bajo eltopo.thl.

Beaconing -- filtrar por "beacon":

```text
grep beacon dns.log

Query: A? 1.beacon.c2.eltopo.thl
Query: A? 2.beacon.c2.eltopo.thl
Query: A? 3.beacon.c2.eltopo.thl
```

1.beacon.c2.eltopo.thl es la primera del grupo -- responde la pregunta 3.

Exfiltracion -- filtrar por "data.":

```text
grep "data\." dns.log | head -3

Query: A? OTk5Ojc6OjoK.data.eltopo.thl
Query: A? Oio6MTgwMDA6MDo5OTk5OTo3Ojo6CmRhZW1vbjoqOjE4MDAwOjA6OTk5OTk6.data.eltopo.thl
Query: A? cm9vdDokNiRzYWx0eSRULlVWcy4uLjoxODAwMDowOjk5OTk5Ojc6OjoKYmlu.data.eltopo.thl
```

La etiqueta mas a la izquierda de cada consulta es base64. Decodificando la tercera con Python:

```text
python3 -c "import base64; print(base64.b64decode('cm9vdDokNiRzYWx0eSRULlVWcy4uLjoxODAwMDowOjk5OTk5Ojc6OjoKYmlu=='))"

b'root:$6$salty$T.UVs...:18000:0:99999:7:::\nbin'
```

Coincide caracter a caracter con etc_shadow_artifact.txt: el atacante trocea /etc/shadow, lo codifica en base64 y lo exfiltra como consultas DNS a subdominios de data.eltopo.thl. La pregunta pide el dominio raiz "sin subdominios de datos": eltopo.thl -- responde la pregunta 4.

## 7. ftp.log -- pivoting y exfiltracion de fichero

Solo 638 bytes -- la sesion completa cabe entera en el archivo:

```text
cat ftp.log

[09:10:13] 192.168.1.10 -> 10.0.0.50 FTP 220 (vsFTPd 3.0.3)
[09:10:13] 192.168.1.10 -> 10.0.0.50 FTP USER devuser
[09:10:13] 10.0.0.50 -> 192.168.1.10 FTP 331 Please specify the password.
[09:10:13] 192.168.1.10 -> 10.0.0.50 FTP PASS developer123
[09:10:13] 10.0.0.50 -> 192.168.1.10 FTP 230 Login successful.
[09:10:13] 192.168.1.10 -> 10.0.0.50 FTP LIST
[09:10:13] 10.0.0.50 -> 192.168.1.10 FTP 226 Directory send OK.
[09:10:13] 192.168.1.10 -> 10.0.0.50 FTP GET client_database_backup.zip
[09:10:13] 10.0.0.50 -> 192.168.1.10 FTP 150 Opening BINARY mode data connection.
[09:10:13] 10.0.0.50 -> 192.168.1.10 FTP 226 Transfer complete.
```

Lectura: el servidor ya comprometido (192.168.1.10) pivota hacia 10.0.0.50 por FTP (pregunta 5), con el usuario devuser (pregunta 6) y la contrasena developer123 (pregunta 7), y descarga client_database_backup.zip (pregunta 8).

## 8. Lecciones tecnicas

- Cuando el objetivo en vivo no expone nada por red, la evidencia puede estar dentro de la propia VM descargable -- vale la pena revisar si el reto ofrece un .ova/.zip.

- 7z extrae particiones de un .vmdk streamOptimized directamente, sin necesidad de qemu-img/qemu-nbd.

- debugfs -R "ls -l ..." y "rdump ..." permiten explorar y volcar un filesystem ext4 SIN montarlo y SIN privilegios de root -- muy util cuando no hay sudo a mano.

- En logs con ruido sintetico de proposito, filtrar por las palabras clave del enunciado (upload.php, .thl, beacon, data.) es mucho mas rapido que leer linea por linea.

- Verificar siempre un hallazgo con una segunda fuente cuando se pueda: aqui, decodificar el base64 del DNS y compararlo byte a byte contra etc_shadow_artifact.txt confirmo que la lectura del log era correcta.

- No confundir la pagina del objetivo tecnico (la IP a analizar) con la pagina de la plataforma donde se responde el cuestionario -- son URLs distintas.
