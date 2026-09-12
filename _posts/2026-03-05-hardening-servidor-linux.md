---
title: "Hardening básico de un servidor Linux"
excerpt: "Lista de medidas mínimas que aplico a todo servidor antes de exponerlo: SSH, firewall, actualizaciones y monitoreo."
date: 2026-03-05
categories:
  - Servidores Linux
tags:
  - linux
  - hardening
  - ssh
  - firewall
---

## SSH

```bash
# /etc/ssh/sshd_config
PermitRootLogin no
PasswordAuthentication no
AllowUsers tuusuario
```

## Firewall

```bash
ufw default deny incoming
ufw allow 22/tcp
ufw enable
```

## Actualizaciones y auditoría

```bash
apt update && apt upgrade -y
lynis audit system
```

## Checklist final

- [ ] Usuario sin privilegios + sudo
- [ ] Claves SSH en lugar de contraseñas
- [ ] Fail2ban activo
- [ ] Respaldos verificados
