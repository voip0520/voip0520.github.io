---
title: "Wanna Some Cookies - Manejo de sesión (hackrocks)"
permalink: /writeups/wanna-some-cookies/
excerpt: "Hackrocks - Wanna Some CookiesDocumento técnico en formato CherryTree con la resolución estructurada del reto, centrado en análisis de cookies, manejo de sesión y obtención de acce"
date: 2026-07-18
categories:
  - Máquinas
tags:
  - hackrocks
  - web
  - cookies
  - sesion
---

> **Plataforma:** hackrocks

Hackrocks - Wanna Some CookiesDocumento técnico en formato CherryTree con la resolución estructurada del reto, centrado en análisis de cookies, manejo de sesión y obtención de acceso a la zona administrativa.

## 1. Datos generales

Reto: Wanna Some CookiesPlataforma: HackrocksCategoría: Web / Session Handling / Cookies

Objetivo: obtener acceso a funcionalidades administrativas identificando debilidades en la gestión de cookies y sesiones.
Resultado obtenido: se logró acceso efectivo al reto trabajando con la sesión del usuario cookieChef tras eliminar la sesión administrativa previa.

## 2. Escenario observado

Durante el análisis en navegador se observó que la aplicación almacenaba múltiples cookies asociadas al flujo de autenticación y estado de usuario. Entre las más relevantes: - access - refresh - Rol - Sesion - session_id - Signature - Status - User - PasswordValores destacados vistos en el entorno: - User = cookieChef - Rol = admin - Status = Loged - Password = doYouLikeRaisins - Sesion = Session_Rasin_2019_02_10Al intentar acceder directamente a la ruta administrativa, el sistema respondió con el mensaje: - Hey! Nothing to see here.Eso indicaba que no bastaba con modificar un solo valor visible; el contexto de sesión también influía en la validación.

## 3. Hipótesis iniciales

Se plantearon varias hipótesis razonables durante la resolución: 
1. La aplicación confiaba parcialmente en cookies del lado cliente para determinar rol o identidad. 
2. Existía una sesión previa inconsistente que impedía que los cambios surtieran efecto. 
3. La sesión del usuario cookieChef era la que realmente correspondía al flujo válido del reto. 
4. El acceso al área administrativa no dependía solo de Rol=admin, sino de una combinación coherente entre usuario, estado y sesión.

## 4. Metodología aplicada

Herramientas utilizadas: 
navegador web, DevTools, panel Application/Storage, Cookie-Editor.
Proceso general: 
a) Inspección del tráfico HTTP y respuesta del servidor.
 b) Enumeración de cookies activas del dominio del reto. 
 c) Revisión de nombres y valores con significado contextual. 
 d) Validación manual de qué cookie o sesión afectaba el acceso. 
 e) Limpieza de sesión inconsistente y reutilización de la sesión correcta.

## 5. Procedimiento de resolución

Paso 1. Acceso inicial al reto
Se abrió la página del desafío y se verificó que el comportamiento estaba vinculado al uso de cookies de sesión.
Paso 2. Inspección de cookies
Desde DevTools / Application / Cookies y Cookie-Editor se identificaron cookies con relación directa a autenticación, usuario y estado.
Paso 3. Detección de información relevante
Se observó el usuario cookieChef y varios valores que sugerían lógica de autorización del lado cliente o sesiones reutilizadas.
Paso 4. Identificación del problema real 
La sesión administrativa previamente presente generaba conflicto con el flujo esperado del reto. Aunque había valores interesantes, el acceso seguía fallando con el mensaje “Nothing to see here”.
Paso 5. Acción correctiva
Se eliminó la sesión de admin y se inició el flujo con la sesión del usuario cookieChef.
Paso 6. Validación
Tras limpiar la sesión conflictiva y trabajar con la sesión válida del usuario cookieChef, el reto permitió continuar y se obtuvo la resolución satisfactoria.

## 6. Evidencia técnica

Indicadores técnicos observados durante la investigación: 
-Existencia de cookies con nombres semánticos (Rol, User, Status, Password, Sesion). 
- Persistencia de una sesión previa que afectaba el comportamiento del endpoint /admin. 
- Respuesta negativa del servidor mientras la combinación de sesión no era coherente. 
- Éxito al eliminar la sesión administrativa y reestablecer el flujo con cookieChef.

## 7. Vulnerabilidad identificada

Tipo principal: gestión insegura de sesión / confianza excesiva en datos de cliente.
Relación con OWASP:
Broken Access Control / Identification and Authentication Failures.El reto demuestra que el control de acceso y el manejo de sesiones pueden debilitarse cuando el cliente conserva información sensible o estados que influyen en la autorización. 
La resolución confirma que la sesión activa era un factor determinante.

## 8. Impacto de seguridad

En un entorno real, una debilidad de este tipo podría permitir: 
- reutilización de sesiones ajenas; 
- persistencia de privilegios indebidos; 
- acceso no autorizado a áreas restringidas; 
- fuga de información sobre roles, estados o usuarios válidos; 
- confusión de contexto entre múltiples sesiones en el mismo navegador.

## 9. Recomendaciones

Medidas recomendadas para una aplicación real: 
1. Validar autorización únicamente del lado servidor. 
2. No exponer información sensible o lógica de autorización en cookies legibles por el cliente. 
3. Invalidar sesiones antiguas o conflictivas al iniciar un nuevo flujo autenticado. 
4. Emplear atributos de seguridad en cookies: HttpOnly, Secure y SameSite. 
5. Asociar sesión, usuario y rol a un contexto consistente validado en backend. 
6. Auditar eventos de autenticación, cambio de sesión y acceso a rutas administrativas.

## 10. Conclusión

La resolución del reto se logró al comprender que el problema no era únicamente modificar cookies visibles, sino identificar y corregir el contexto de sesión activo. 
El punto decisivo fue eliminar la sesión de admin e iniciar con la sesión del usuario cookieChef, lo que permitió completar el reto correctamente.



Captura sin sesion_rasin





Iniciamos sesion con el usuario 

User = cookieChef
Password = doYouLikeRaisins



Y se obtiene el acceso


Reto superado
![captura 1](/assets/informes/wanna-some-cookies/img01.jpg)

![captura 2](/assets/informes/wanna-some-cookies/img02.jpg)

![captura 3](/assets/informes/wanna-some-cookies/img03.jpg)

![captura 4](/assets/informes/wanna-some-cookies/img04.jpg)


## 11. Resumen ejecutivo

Se resolvió el reto Wanna Some Cookies analizando cookies de autenticación en navegador. 
Tras revisar valores como Rol, User, Status, Password y Sesion, se determinó que una sesión administrativa previa interfería con la validación. 
La resolución exitosa consistió en eliminar la sesión de admin y operar con la sesión de cookieChef. 
Esto evidenció una debilidad asociada a gestión insegura de sesión y control de acceso dependiente del contexto cliente-servidor.
