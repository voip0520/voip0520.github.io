---
title: "Shells & Enumeración: guía práctica"
date: 2026-09-24
layout: single
permalink: /notas/shells-enumeracion/
author_profile: true
classes: wide
categories: [Notas, Pentesting]
tags: [Shell, Reverse-Shell, Web-Shell, Bind-Shell, Windows, Linux, Enumeracion]
excerpt: "Fundamentos de shells y checklist de enumeración para laboratorios autorizados."
toc: true
toc_label: "Contenido"
toc_sticky: false
---

> **Uso autorizado:** estos apuntes están orientados a laboratorios, CTF y evaluaciones de seguridad con autorización.

## 1. ¿Qué es una Shell?

Una **shell** es una interfaz que permite interactuar con el sistema operativo mediante comandos. En seguridad ofensiva, obtener una shell durante un laboratorio permite ejecutar acciones con los permisos del usuario o proceso comprometido.

Ejemplos comunes son `cmd.exe` y PowerShell en Windows, y `sh` o `bash` en Linux.

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

## 3. ¿Qué es una Web Shell?

Una **web shell** es código alojado en un servidor web que permite ejecutar determinadas acciones o comandos a través de una aplicación web.

Su presencia puede estar asociada a una carga de archivos insegura, una aplicación comprometida o una configuración incorrecta.

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

## 5. Reverse Shell vs Bind Shell

| Característica | Reverse Shell | Bind Shell |
|---|---|---|
| Inicia la conexión | Sistema objetivo | Operador |
| Escucha | Equipo del operador | Sistema objetivo |
| Dirección inicial | Objetivo → operador | Operador → objetivo |
| Requiere puerto accesible en objetivo | Normalmente no | Sí |
| Puede verse afectada por firewall/NAT | Sí | Sí |

## 6. Enumeración de Windows

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

## 7. Enumeración de Linux

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

## 8. Checklist de enumeración

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

## 9. Perspectiva Blue Team

Las shells también pueden estudiarse desde defensa. Algunos indicadores que merecen investigación son:

- conexiones salientes inesperadas desde servidores;
- shells iniciadas por procesos de aplicaciones web;
- intérpretes de comandos ejecutados por cuentas de servicio;
- nuevos puertos en escucha;
- ejecución anómala de PowerShell, `cmd`, `bash` o `sh`;
- archivos ejecutables o scripts inesperados en directorios web.

Una alerta aislada no confirma necesariamente una intrusión; debe correlacionarse con usuario, proceso padre, destino de red, hora, activo y actividad previa.

## 10. Flujo de estudio

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

## 11. Conclusión

Comprender la diferencia entre **shell**, **reverse shell**, **web shell** y **bind shell** ayuda tanto en pentesting como en detección defensiva. La enumeración posterior debe realizarse de forma ordenada para entender el contexto del sistema antes de evaluar posibles vectores de escalada o movimiento dentro de un laboratorio autorizado.
