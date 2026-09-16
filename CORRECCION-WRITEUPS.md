# Corrección de Máquinas y Writeups

Este paquete corrige el problema de tarjetas sin estilo y activa la plantilla profesional en los posts.

Cambios principales:
- CSS de `.machine-grid` y `.machine-card` integrado directamente en `assets/css/main.scss`.
- CSS de la cabecera profesional de writeups integrado en `main.scss`.
- `_layouts/writeup.html` reutilizable.
- `_includes/machine-card.html` corregido.
- `_pages/maquinas.md` conserva las agrupaciones por plataforma.
- Posts configurados con `layout: writeup`; el contenido original se conserva.

Para publicar, reemplaza el contenido del repositorio local con esta versión, revisa GitHub Desktop y haz Commit + Push.
