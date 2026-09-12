# Hacking 0520 — portafolio estilo "Hack Notes"

Jekyll + tema **Minimal Mistakes** con skin **neon** (negro con acentos rosa),
barra lateral de autor, breadcrumbs, "X minute read" y archivos automáticos por
categoría, etiqueta y año. Ya viene configurado con tus datos
(Oswaldo Quishpe / voip0520 / LinkedIn / avatar de GitHub).

---

## Importante sobre tu repositorio actual

Hoy tienes `voip0520/hacking0520.github.io`, que es un **fork del tema Chirpy**.
Dos problemas:

1. Un repo se publica en `https://USUARIO.github.io` solo si se llama igual que
   tu usuario. Como el tuyo se llama `hacking0520.github.io` pero tu usuario es
   `voip0520`, la URL te queda
   `https://voip0520.github.io/hacking0520.github.io/`.
2. Al ser un fork del tema, arrastra todo el historial y los archivos de Chirpy,
   lo que hace más difícil mantenerlo.

**Recomendación:** crea un repo nuevo y vacío llamado exactamente
`voip0520.github.io` y sube ahí estos archivos. El sitio quedará en
`https://voip0520.github.io`.

Si prefieres seguir usando el repo actual, cambia en `_config.yml`:

```yaml
url     : "https://voip0520.github.io"
baseurl : "/hacking0520.github.io"
```

---

## 1. Subir el sitio

```bash
cd portafolio
git init
git add .
git commit -m "Portafolio con Minimal Mistakes"
git branch -M main
git remote add origin https://github.com/voip0520/voip0520.github.io.git
git push -u origin main
```

## 2. Activar GitHub Pages

Repo → **Settings → Pages → Source: Deploy from a branch** → rama `main`,
carpeta `/ (root)` → **Save**. En 1-2 minutos queda publicado.

## 3. Qué falta que revises

- [ ] El nombre real de tu certificación INE (está como "Certificación INE" en
      `_config.yml`, `_pages/about.md` y el post de review).
- [ ] Tu correo, si quieres mostrarlo: descomenta el bloque `Email` en
      `_config.yml`.
- [ ] Completa `_pages/about.md` con tu experiencia laboral.
- [ ] Reemplaza los 4 posts de ejemplo por contenido real (o bórralos).

## 4. Publicar un artículo

Crea un archivo en `_posts/` llamado `AAAA-MM-DD-titulo.md`:

```markdown
---
title: "Título del artículo"
excerpt: "Resumen corto que aparece en los listados."
date: 2026-04-01
categories:
  - Telefonía IP
tags:
  - asterisk
  - sip
---

Contenido en Markdown...
```

La página `/categories/` con los contadores (Redes 2, Telefonía IP 3, ...) se
genera sola a partir del campo `categories`. Las imágenes van en
`assets/images/` y se insertan con `![texto](/assets/images/archivo.png)`.

## 5. (Opcional) Probarlo en tu PC antes de subirlo

```bash
sudo apt install ruby-full build-essential zlib1g-dev
gem install bundler
bundle install
bundle exec jekyll serve
# http://localhost:4000
```

---

### Estructura

```
_config.yml              configuración y barra lateral
_data/navigation.yml     menú superior
_pages/                  Acerca de, Categorías, Etiquetas, Artículos
_posts/                  tus artículos
assets/images/           imágenes
index.html               portada
```

### Skins disponibles
`air`, `aqua`, `contrast`, `dark`, `dirt`, `mint`, `neon`, `plum`, `sunrise`.
