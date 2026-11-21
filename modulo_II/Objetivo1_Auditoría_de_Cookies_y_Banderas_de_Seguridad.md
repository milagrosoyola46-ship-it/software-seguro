# Objetivo 1: Auditoría de Cookies y Banderas de Seguridad

**Herramienta:** DevTools del navegador (Chrome) → Application → Cookies
**Sitios analizados:** github.com, Classroom, Instagram

## GitHub (github.com)
 foto

Cookie de sesión principal: **`user_session`** (también `logged_in`, `_gh_sess`).

| Cookie | HttpOnly | Secure | SameSite |
|---|---|---|---|
| user_session | ✓ | ✓ | Lax |
| logged_in | ✓ | ✓ | Lax |
| _gh_sess | ✓ | ✓ | Strict |
| dotcom_user | ✗ | ✓ | Lax |

**Observación:** las cookies de sesión/autenticación están correctamente protegidas con `HttpOnly` y `Secure`. La cookie `dotcom_user` (solo guarda el username, no es la de sesión) no tiene `HttpOnly`, pero el riesgo es menor al no ser un token de autenticación.

## 2. Google / Classroom (google.com)

📸 ACÁ VA: `google_classroom_cookies.png`

| Cookie | HttpOnly | Secure | SameSite |
|---|---|---|---|
| SID | ✗ | ✓ | — |
| APISID / SAPISID | ✗ | ✓ | — |
| _Secure-1PSID / 3PSID | ✓ | ✓ | — |
| HSID, SSID, SIDCC | ✓ | ✓ | — |

**Observación:** todas las cookies tienen `Secure` activada. Sin embargo, `SID`, `APISID` y `SAPISID` no tienen `HttpOnly` — esto es intencional en Google, ya que son usadas por JavaScript del lado del cliente para firmar pedidos a sus APIs (SAPISIDHASH). Es un caso donde la falta de `HttpOnly` es una decisión de diseño, no un descuido, pero igualmente amplía la superficie de ataque ante un XSS.

## 3. Instagram (instagram.com)

📸 ACÁ VA: `instagram_cookies.png`

| Cookie | HttpOnly | Secure | SameSite |
|---|---|---|---|
| sessionid | ✓ | ✓ | None |
| ds_user_id | ✗ | ✗ | None |
| csrftoken | ✗ | ✓ | — |
| ig_did | ✓ | ✓ | — |

**Observación:** la cookie de sesión real, `sessionid`, está bien protegida (HttpOnly + Secure). Pero `ds_user_id` (que guarda el ID numérico de usuario) **no tiene ninguna de las dos banderas activadas** — queda accesible por JavaScript y se podría transmitir por HTTP sin cifrar.

## 4. ¿Qué protege la bandera HttpOnly?

La bandera `HttpOnly` evita que la cookie sea accesible desde JavaScript del lado del cliente (`document.cookie` no la devuelve). Esto protege específicamente contra ataques de **XSS (Cross-Site Scripting)**: si un atacante logra inyectar un script malicioso en la página, ese script no va a poder leer ni robar el valor de la cookie de sesión para enviarlo a un servidor externo (robo de sesión / session hijacking).

## 5. ¿Qué riesgo corre el usuario si la cookie de sesión no tiene Secure activada?

La bandera `Secure` obliga al navegador a enviar la cookie **únicamente por HTTPS**, nunca por HTTP plano. Si una cookie de sesión no tiene esta bandera:
- Puede ser interceptada en una red no confiable (ej: un Wi-Fi público) mediante un ataque de **Man-in-the-Middle**, ya que viajaría sin cifrar si el sitio llega a ser accedido (o redirigido) por HTTP.
- Un atacante que capture esa cookie puede reutilizarla para **suplantar la sesión del usuario** sin necesidad de conocer su contraseña.

## 6. Conclusión

De los tres sitios auditados, las cookies de sesión/autenticación propiamente dichas (`user_session` en GitHub, `sessionid` en Instagram, los tokens `_Secure-*` de Google) están correctamente protegidas con `HttpOnly` y `Secure`. El mayor riesgo encontrado es puntual: cookies secundarias (`ds_user_id` en Instagram) que, sin ser el token de sesión, exponen información identificatoria del usuario sin ninguna de las dos protecciones.
