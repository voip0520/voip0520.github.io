---
title: "Metodología de Pentesting"
permalink: /metodologia/
author_profile: true
toc: true
toc_sticky: true
classes: wide
---

<p class="page-lead">Flujo de trabajo que utilizo para documentar evaluaciones y laboratorios autorizados. La metodología prioriza alcance, evidencia reproducible y recomendaciones de mitigación.</p>

<div class="method-flow">
<div><b>01</b><strong>Alcance</strong><span>Objetivos · autorización · reglas</span></div>
<div><b>02</b><strong>Reconocimiento</strong><span>Activos · superficie expuesta</span></div>
<div><b>03</b><strong>Scanning</strong><span>Puertos · servicios · versiones</span></div>
<div><b>04</b><strong>Enumeración</strong><span>Usuarios · recursos · aplicaciones</span></div>
<div><b>05</b><strong>Validación</strong><span>Vulnerabilidades · impacto</span></div>
<div><b>06</b><strong>Explotación</strong><span>Prueba controlada</span></div>
<div><b>07</b><strong>Post-explotación</strong><span>Privilegios · alcance real</span></div>
<div><b>08</b><strong>Reporte</strong><span>Evidencia · riesgo · remediación</span></div>
</div>

## 1. Alcance y reglas de compromiso
Antes de ejecutar pruebas se definen activos autorizados, exclusiones, ventanas de trabajo, objetivos y criterios de detención. Esto evita afectar sistemas fuera del alcance.

## 2. Reconocimiento y descubrimiento
Se identifica la superficie de ataque y los servicios accesibles. Herramientas habituales: **Nmap**, consultas DNS y revisión manual de aplicaciones.

## 3. Enumeración
Se profundiza sobre los servicios encontrados para identificar versiones, configuraciones, usuarios, recursos compartidos y posibles rutas de acceso.

## 4. Análisis y validación
Los hallazgos se contrastan con su contexto técnico. Una versión antigua no se considera automáticamente explotable: se valida la condición y su impacto dentro del laboratorio.

## 5. Explotación controlada
Cuando el alcance lo permite, se demuestra el impacto con la mínima acción necesaria. La evidencia debe ser reproducible y evitar cambios innecesarios sobre el objetivo.

## 6. Escalada y post-explotación
Se revisan permisos, configuraciones y rutas de escalada para determinar el impacto real del compromiso, siempre dentro de las reglas del entorno.

## 7. Reporte y remediación
Cada hallazgo debería incluir activo, evidencia, impacto, causa, severidad contextual y una recomendación técnica. Cuando aplica, relaciono la actividad con **MITRE ATT&CK** para facilitar su comprensión defensiva.

### Ejemplo de mapeo MITRE ATT&CK

| Técnica | Uso documental |
|---|---|
| T1046 — Network Service Scanning | Descubrimiento de servicios de red |
| T1190 — Exploit Public-Facing Application | Validación de una aplicación expuesta vulnerable |
| T1059 — Command and Scripting Interpreter | Uso de intérpretes durante una prueba controlada |

> Las técnicas se añaden únicamente cuando describen realmente la actividad observada; no se asignan de forma automática a todos los writeups.
