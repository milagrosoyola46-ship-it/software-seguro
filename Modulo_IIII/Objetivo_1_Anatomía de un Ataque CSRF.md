# Objetivo 1: Anatomía de un Ataque CSRF (Cross-Site Request Forgery)

## ¿Qué es un ataque CSRF?

**CSRF (Cross-Site Request Forgery)**, o "falsificación de petición en sitios cruzados", es un ataque que **engaña al navegador de una víctima autenticada para que envíe, sin que ella se dé cuenta, una petición HTTP no deseada hacia un sitio en el que sí tiene sesión iniciada**. El atacante no necesita robar la contraseña ni la sesión de la víctima: le alcanza con conseguir que su navegador, que ya está autenticado, dispare la petición por ella.

La clave del ataque no está en vulnerar la autenticación, sino en abusar de un comportamiento *legítimo y esperado* de los navegadores: el envío automático de cookies.

---

## El flujo exacto de un ataque CSRF exitoso

### Paso 1 — La víctima inicia sesión en el sitio legítimo

La víctima se loguea normalmente en, por ejemplo, `banco.com`. El servidor responde con una cookie de sesión:

```
Set-Cookie: session_id=abc123; Path=/
```

El navegador guarda esa cookie y, a partir de ahí, **la va a adjuntar automáticamente en cada petición futura hacia `banco.com`**, sin que ninguna página ni script tenga que pedírselo explícitamente. Esto es simplemente cómo funcionan las cookies por diseño.

### Paso 2 — La víctima visita un sitio controlado por el atacante (sin cerrar sesión en el primero)

Sin desloguearse de `banco.com`, la víctima visita `sitio-malicioso.com` — puede llegar ahí por un link en un email, un mensaje, un banner publicitario, etc.

### Paso 3 — El sitio malicioso dispara una petición hacia el sitio legítimo

La página maliciosa contiene algo tan simple como:

```html
<form action="https://banco.com/transferir" method="POST" id="formMalicioso">
  <input type="hidden" name="cuentaDestino" value="CUENTA_DEL_ATACANTE">
  <input type="hidden" name="monto" value="5000">
</form>
<script>
  document.getElementById('formMalicioso').submit();
</script>
```

O, de forma más sigilosa, con una imagen invisible (para peticiones GET):

```html
<img src="https://banco.com/transferir?cuentaDestino=ATACANTE&monto=5000" style="display:none">
```

El navegador de la víctima ejecuta ese `submit()` (o carga esa imagen) automáticamente, sin que la víctima haga clic en nada ni se entere.

### Paso 4 — El navegador adjunta la cookie de sesión automáticamente

Acá está el corazón de la falla: cuando el navegador arma la petición hacia `banco.com/transferir`, revisa qué cookies tiene guardadas para ese dominio y las agrega **sin importar desde qué sitio se originó la petición**. El navegador no distingue "¿esta petición la generó una página de confianza de `banco.com`, o una página externa maliciosa?" — solo le importa el dominio de destino.

Entonces la petición maliciosa sale con la cookie de sesión válida:

```
POST /transferir HTTP/1.1
Host: banco.com
Cookie: session_id=abc123
Content-Type: application/x-www-form-urlencoded

cuentaDestino=CUENTA_DEL_ATACANTE&monto=5000
```

### Paso 5 — El servidor procesa la petición como legítima

`banco.com` recibe la petición, ve una cookie de sesión válida (`session_id=abc123`), confirma que corresponde a una sesión autenticada de la víctima, y **ejecuta la transferencia** — porque, desde su perspectiva, no hay forma de distinguir esta petición de una que la víctima hubiera enviado voluntariamente desde su propia página. El servidor confió únicamente en "¿tiene una cookie de sesión válida?", sin verificar "¿esta petición realmente se originó en mi propio sitio, por una acción intencional del usuario?".

---

## ¿Por qué ocurre esta falla? El comportamiento automático de las cookies

La causa raíz de CSRF es una combinación de dos comportamientos, cada uno razonable por separado, pero peligrosos juntos:

1. **Las cookies se envían automáticamente en cada petición al dominio al que pertenecen**, sin importar el origen de la petición (si vino de una pestaña del propio sitio, de un link externo, de un formulario en otra página, de una imagen embebida, etc.). El navegador no pregunta "¿quién pidió esto?", solo "¿a qué dominio va dirigido?".

2. **El servidor, por defecto, solo valida la presencia de una cookie de sesión como prueba de autenticación**, sin verificar el origen real de la petición (por ejemplo, mediante un token anti-CSRF, o validando el header `Origin`/`Referer`).

Esta combinación hace que **cualquier sitio en internet pueda "hablar" en nombre del navegador autenticado de la víctima**, simplemente haciendo que el navegador apunte una petición hacia el sitio vulnerable. El atacante nunca ve ni necesita la cookie — ni siquiera sabe cuál es: el navegador de la víctima se encarga de adjuntarla.

Esto es distinto de un ataque como XSS: en CSRF el atacante no ejecuta código dentro del dominio víctima ni roba datos directamente; simplemente **aprovecha la confianza implícita que el servidor deposita en la cookie**, sin que exista ninguna prueba real de que la petición representa la intención del usuario.

---

## Cómo el atributo de cookie `SameSite` mitiga radicalmente este vector

El atributo `SameSite` se agrega al momento de setear la cookie:

```
Set-Cookie: session_id=abc123; SameSite=Strict
```

Y le indica al **navegador** (no al servidor) bajo qué condiciones puede enviar esa cookie en peticiones que se originan desde un sitio distinto al que la emitió (peticiones "cross-site"). Ataca directamente la causa raíz: el envío automático e incondicional de la cookie.

### `SameSite=Strict`

- La cookie **nunca se envía** en ninguna petición cross-site, sin excepciones — ni siquiera si el usuario hace clic en un link legítimo desde otro sitio hacia el sitio protegido.
- Con esto, el escenario del Paso 4 del ataque descripto arriba **no puede ocurrir**: cuando `sitio-malicioso.com` dispara la petición hacia `banco.com`, el navegador directamente **no adjunta** la cookie de sesión, porque la petición es cross-site. El servidor recibe la petición sin cookie de sesión válida, la trata como no autenticada, y la rechaza.
- Es la protección más fuerte, pero tiene un costo de usabilidad: si la víctima llega a `banco.com` haciendo clic en un link desde un email o desde otro sitio, **tampoco estará logueada** en esa primera carga (porque esa navegación inicial también es cross-site), obligándola a volver a loguearse.

### `SameSite=Lax`

- Es el valor **por defecto** en los navegadores modernos si no se especifica el atributo.
- Es un punto intermedio: la cookie **no se envía en peticiones cross-site "de fondo"** (como las que dispara un formulario enviado por JavaScript, una imagen, un `fetch`/XHR, o un formulario con método POST) — que es exactamente el tipo de petición que usa un ataque CSRF típico como el del ejemplo.
- Sí permite el envío de la cookie en **navegaciones de nivel superior con método GET** (por ejemplo, cuando el usuario hace clic en un link normal que lo lleva a `banco.com`), preservando la experiencia de "llegar ya logueado" al hacer clic en un enlace legítimo.
- Esto mitiga la gran mayoría de los ataques CSRF reales (que casi siempre dependen de un POST automático disparado por JavaScript o un formulario oculto), sin sacrificar tanto la usabilidad como `Strict`.

### Por qué esto mitiga el ataque de forma radical

Volviendo al flujo del ataque: en el Paso 3, la petición maliciosa hacia `banco.com/transferir` se origina desde `sitio-malicioso.com` — es decir, es una petición **cross-site** por definición. Si la cookie de sesión tiene `SameSite=Lax` o `SameSite=Strict`:

- Con `Strict`: la cookie jamás se adjunta en esa petición cross-site, sea GET o POST.
- Con `Lax`: como la petición del formulario es un **POST** disparado automáticamente (no una navegación de nivel superior iniciada por el usuario con GET), la cookie **tampoco se adjunta**.

En ambos casos, la petición maliciosa llega al servidor **sin la cookie de sesión**. El servidor ya no puede asociarla a la sesión de la víctima, la trata como anónima/no autenticada, y la rechaza (o redirige a un login). El ataque falla en el punto exacto donde antes tenía éxito: el navegador deja de hacer el trabajo sucio de "autenticar" automáticamente una petición que en realidad nunca fue intención del usuario.

Es importante notar que `SameSite` **no reemplaza** otras defensas recomendadas contra CSRF (como tokens anti-CSRF únicos por sesión/formulario, o validar el header `Origin`), pero sí elimina la causa raíz más común y sencilla de explotar, por lo que se considera hoy una mitigación de primera línea, ampliamente soportada por los navegadores modernos.

---

## Resumen

| Elemento | Descripción |
|---|---|
| **Causa raíz** | El navegador adjunta cookies automáticamente en cualquier petición hacia el dominio dueño de la cookie, sin importar el origen de la petición; el servidor confía solo en la presencia de esa cookie para autenticar. |
| **Vector típico** | Formulario auto-enviado o imagen oculta en un sitio malicioso, apuntando a una acción sensible del sitio legítimo (transferencia, cambio de contraseña, cambio de email, etc.). |
| **Por qué funciona** | El navegador de la víctima, ya autenticado, hace de intermediario involuntario: arma la petición con la cookie de sesión válida sin que la víctima lo note. |
| **Mitigación clave** | Atributo de cookie `SameSite`. |
| **`SameSite=Strict`** | Bloquea el envío de la cookie en absolutamente toda petición cross-site. Máxima seguridad, mayor impacto en usabilidad. |
| **`SameSite=Lax`** | Bloquea el envío en peticiones cross-site "de fondo" (POST, fetch, imágenes, iframes), pero permite el envío en navegaciones GET de nivel superior iniciadas por el usuario. Buen balance seguridad/usabilidad; es el valor por defecto actual. |
