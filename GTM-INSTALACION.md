# Google Tag Manager / GA4

Contenedor GTM: `GTM-NQWGLRHN`

El contenedor se carga globalmente mediante `_includes/head/custom.html`, hook soportado por Minimal Mistakes.

En Google Tag Manager, la etiqueta GA4 configurada es:
- Tipo: Etiqueta de Google
- ID: `G-BCXTFNGQ4E`
- Activador: `Initialization - All Pages`

## Después de subir a GitHub
1. Esperar a que GitHub Pages termine el despliegue.
2. En GTM, usar **Vista previa** con `https://voip0520.github.io`.
3. Confirmar que aparece el contenedor `GTM-NQWGLRHN` y que dispara `Google Analytics - Portafolio`.
4. Volver a GTM y pulsar **Enviar** para publicar el contenedor.
5. En GA4, abrir **Tiempo real** y visitar el portafolio desde otro dispositivo/red para comprobar la recepción.

Nota: no se añadió `gtag.js` directamente al sitio para evitar duplicar la medición; GA4 se administra desde GTM.
