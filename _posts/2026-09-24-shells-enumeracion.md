---
title: "Shells, Enumeración & PrivEsc: guía práctica"
date: 2026-09-24
layout: single
permalink: /notas/shells-enumeracion/
author_profile: true
classes: wide
categories: [Notas, Pentesting]
tags: [Shell, Reverse-Shell, Web-Shell, Bind-Shell, Windows, Linux, Enumeracion]
excerpt: "Shells, enumeración y fundamentos de escalada de privilegios para laboratorios autorizados."
toc: true
toc_label: "Contenido"
toc_sticky: false
---

> **Uso autorizado:** estos apuntes están orientados a laboratorios, CTF y evaluaciones de seguridad con autorización.



## 1. ¿Qué es una Shell?

Una **shell** es una interfaz que permite interactuar con el sistema operativo mediante comandos. En seguridad ofensiva, obtener una shell durante un laboratorio permite ejecutar acciones con los permisos del usuario o proceso comprometido.

Ejemplos comunes son `cmd.exe` y PowerShell en Windows, y `sh` o `bash` en Linux.

![Ejemplo de una shell local en Kali Linux](/assets/images/notas/shells/01-shell.png)

*Figura 1. Ejemplo visual de una shell local y consulta de identidad del usuario.*

## 2. ¿Qué es una Reverse Shell?

En una **reverse shell**, el sistema remoto inicia una conexión hacia el equipo del operador y entrega una sesión de comandos.

```text
Equipo del operador
        ↑
        │ conexión iniciada desde el objetivo
        │
Sistema objetivo
```

En un laboratorio controlado, normalmente intervienen dos componentes:

```text
Operador: escucha una conexión
Objetivo: inicia la conexión de retorno
```

Es importante revisar la dirección de la conexión, el puerto utilizado, el usuario de la sesión y las restricciones de red existentes.

![Ejemplo de Reverse Shell](/assets/images/notas/shells/02-reverse-shell.png)

*Figura 2. Representación de una Reverse Shell desde el sistema objetivo hacia el equipo del operador.*

## 3. ¿Qué es una Web Shell?

Una **web shell** es código alojado en un servidor web que permite ejecutar determinadas acciones o comandos a través de una aplicación web.

Su presencia puede estar asociada a una carga de archivos insegura, una aplicación comprometida o una configuración incorrecta.

![Ejemplo de Web Shell](/assets/images/notas/shells/04-web-shell.png)

*Figura 4. Ejemplo visual de una Web Shell utilizada dentro de un laboratorio controlado.*

Desde Blue Team conviene revisar:

- archivos nuevos o modificados en el directorio web;
- procesos iniciados por el servidor web;
- conexiones de red inusuales;
- peticiones HTTP sospechosas;
- cambios inesperados en permisos y propietarios.

## 4. ¿Qué es una Bind Shell?

En una **bind shell**, el sistema objetivo abre un puerto y queda escuchando conexiones.

```text
Equipo del operador
        │
        │ conexión hacia el objetivo
        ↓
Sistema objetivo
Puerto en escucha
```

A diferencia de una reverse shell, aquí es el operador quien inicia la conexión hacia el puerto abierto en el objetivo.

![Ejemplo de Bind Shell](/assets/images/notas/shells/03-bind-shell.png)

*Figura 3. Representación de una Bind Shell con el sistema objetivo escuchando conexiones.*

## 5. Reverse Shell vs Bind Shell

| Característica | Reverse Shell | Bind Shell |
|---|---|---|
| Inicia la conexión | Sistema objetivo | Operador |
| Escucha | Equipo del operador | Sistema objetivo |
| Dirección inicial | Objetivo → operador | Operador → objetivo |
| Requiere puerto accesible en objetivo | Normalmente no | Sí |
| Puede verse afectada por firewall/NAT | Sí | Sí |

## 6. Verificación de conexión y acceso

Una vez establecida una shell dentro del laboratorio, se valida el contexto de la sesión antes de continuar con la enumeración.

![Verificación de conexión y acceso](/assets/images/notas/shells/05-conexion-acceso.png)

*Figura 5. Verificación visual de una sesión interactiva y del contexto del sistema.*

## 7. Enumeración de Windows

Después de obtener acceso autorizado a un sistema Windows, la enumeración busca comprender el contexto de la sesión.

### Identidad y privilegios

```powershell
whoami
whoami /all
whoami /priv
```

### Información del sistema

```powershell
hostname
systeminfo
```

### Usuarios y grupos

```powershell
net user
net localgroup
net localgroup administrators
```

### Red

```powershell
ipconfig /all
route print
arp -a
netstat -ano
```

### Procesos y servicios

```powershell
tasklist
sc query
```

### Variables y entorno

```powershell
set
```

El objetivo es identificar el usuario actual, privilegios, versión del sistema, interfaces, rutas, servicios y posibles relaciones con otros activos.

![Enumeración básica en Windows](/assets/images/notas/shells/07-enumeracion-windows.png)

*Figura 7. Ejemplo de recopilación básica de información en Windows.*

## 8. Enumeración de Linux

### Identidad

```bash
whoami
id
groups
```

### Sistema operativo y kernel

```bash
hostname
uname -a
cat /etc/os-release
```

### Red

```bash
ip addr
ip route
ss -tulpen
```

### Usuarios

```bash
cat /etc/passwd
```

### Procesos

```bash
ps aux
```

### Privilegios sudo

```bash
sudo -l
```

### Archivos SUID

```bash
find / -perm -4000 -type f 2>/dev/null
```

### Tareas programadas

```bash
cat /etc/crontab
ls -la /etc/cron.d/
```

Estos datos ayudan a construir una visión del host antes de realizar cualquier validación adicional.

![Enumeración básica en Linux](/assets/images/notas/shells/06-enumeracion-linux.png)

*Figura 6. Ejemplo de información del sistema, red y servicios durante la enumeración en Linux.*

## 9. Escalada de privilegios

La **escalada de privilegios (Privilege Escalation)** consiste en pasar de una cuenta con permisos limitados a otra con mayores privilegios, por ejemplo `root` en Linux o `Administrator/SYSTEM` en Windows. En un laboratorio autorizado, primero se enumera el sistema para identificar configuraciones inseguras, permisos excesivos, credenciales expuestas o software vulnerable.

### PrivEsc en Linux: puntos de revisión

```bash
sudo -l
find / -perm -4000 -type f 2>/dev/null
cat /etc/crontab
ls -la /etc/cron.d/
```

![Escalada de privilegios en Linux](/assets/images/notas/shells/08-privesc-linux.png)

*Figura 8. Revisión de sudo, binarios SUID y tareas programadas en Linux.*

Se revisan permisos `sudo`, binarios SUID, tareas programadas, archivos escribibles y configuraciones que puedan otorgar privilegios superiores. Un hallazgo debe analizarse antes de considerarlo explotable.

### PrivEsc en Windows: puntos de revisión

```powershell
whoami /priv
whoami /groups
sc query
schtasks /query /fo LIST /v
net localgroup administrators
```

La revisión se centra en privilegios del usuario, grupos, servicios, tareas programadas y configuraciones con permisos inseguros.

![Escalada de privilegios en Windows](/assets/images/notas/shells/09-privesc-windows.png)

*Figura 9. Revisión de privilegios y tareas programadas en Windows.*

```text
Shell inicial
     ↓
Enumeración
     ↓
Usuario y privilegios
     ↓
Servicios / procesos
     ↓
Sudo / SUID / tareas programadas
     ↓
Permisos y configuraciones
     ↓
Identificación del vector
     ↓
Validación controlada
     ↓
Documentación
```

> Encontrar un SUID, servicio, tarea programada o privilegio especial no confirma por sí solo una vulnerabilidad; debe validarse dentro del alcance autorizado.

## 10. Checklist de enumeración

```text
[ ] Usuario actual
[ ] Grupos y privilegios
[ ] Hostname
[ ] Sistema operativo
[ ] Versión / kernel
[ ] Interfaces de red
[ ] Rutas
[ ] Puertos en escucha
[ ] Procesos
[ ] Servicios
[ ] Usuarios locales
[ ] Tareas programadas
[ ] Permisos especiales
[ ] Software instalado
[ ] Variables de entorno
[ ] Controles de seguridad
```

## 11. Perspectiva Blue Team

Las shells también pueden estudiarse desde defensa. Algunos indicadores que merecen investigación son:

- conexiones salientes inesperadas desde servidores;
- shells iniciadas por procesos de aplicaciones web;
- intérpretes de comandos ejecutados por cuentas de servicio;
- nuevos puertos en escucha;
- ejecución anómala de PowerShell, `cmd`, `bash` o `sh`;
- archivos ejecutables o scripts inesperados en directorios web.

Una alerta aislada no confirma necesariamente una intrusión; debe correlacionarse con usuario, proceso padre, destino de red, hora, activo y actividad previa.

## 12. Flujo de estudio

```text
Acceso autorizado
      ↓
Identificar la shell
      ↓
Usuario y privilegios
      ↓
Sistema operativo
      ↓
Red
      ↓
Procesos y servicios
      ↓
Usuarios y permisos
      ↓
Configuraciones relevantes
      ↓
Documentar hallazgos
```

## 13. Conclusión

Comprender la diferencia entre **shell**, **reverse shell**, **web shell** y **bind shell** ayuda tanto en pentesting como en detección defensiva. La enumeración posterior debe realizarse de forma ordenada para entender el contexto del sistema antes de evaluar posibles vectores de escalada o movimiento dentro de un laboratorio autorizado.
