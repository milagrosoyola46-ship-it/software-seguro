# Objetivo 1: El Arte de la Evasión de Filtros (WAF Bypass)

## Escenario

Un desarrollador, al enterarse de que XSS permite inyectar `<script>` para ejecutar JavaScript, implementa una defensa que elimina automáticamente toda ocurrencia de las etiquetas `<script>` y `</script>` de los comentarios de los usuarios. La pregunta es: **¿por qué esta defensa no sirve de nada?**

## Por qué esta defensa no sirve

### 1. Confunde "la etiqueta `<script>`" con "la capacidad de ejecutar JavaScript"

El error conceptual de fondo es creer que `<script>` es la *única* forma de que el navegador ejecute JavaScript. En realidad, HTML ofrece **decenas de vectores distintos** para disparar código JavaScript sin necesidad de esa etiqueta puntual:

- **Atributos de evento (event handlers)**: `onerror`, `onload`, `onclick`, `onmouseover`, `onfocus`, `onpointerdown`, etc. Cualquier elemento HTML que soporte estos atributos puede ejecutar JS cuando ese evento ocurre.
- **Esquemas de URL ejecutables**: `href="javascript:..."` en un link, o `src="javascript:..."` en ciertos contextos.
- **Elementos que disparan eventos automáticamente**, sin que el usuario interactúe: una imagen con una URL rota dispara `onerror` sola; un `<svg>` dispara `onload` al renderizarse; un `<body>` dispara `onload`/`onpageshow`.

Filtrar una sola etiqueta (`<script>`) es combatir un síntoma puntual, no la causa raíz (la posibilidad de que el navegador interprete HTML/atributos controlados por el atacante). Es el equivalente de una cerradura que solo bloquea una de las veinte puertas de una casa.

### 2. Es un filtro basado en coincidencia de texto, no en comprensión real del HTML

Un filtro que simplemente "elimina la cadena `<script>`" trabaja a nivel de texto plano, sin parsear ni entender la estructura HTML real. Esto lo hace doblemente débil:

- No contempla variaciones de **mayúsculas/minúsculas** (`<ScRiPt>`), que el navegador interpreta igual mientras el filtro, si no es case-insensitive, puede no detectar.
- No contempla que el atacante puede anidar la etiqueta para sobrevivir a un reemplazo no recursivo: si el filtro reemplaza `<script>` por cadena vacía **una sola vez**, un payload como `<scr<script>ipt>` se convierte, tras ese único reemplazo, en `<script>` válido.
- Y, lo más importante para este ejercicio: **ni siquiera hace falta la palabra "script" en ningún lado**. El atacante puede lograr ejecución de JavaScript usando etiquetas y atributos completamente distintos.

## Dos payloads que ejecutan JavaScript sin usar la etiqueta `<script>`

### Payload 1 — Imagen con `onerror`

```html
<img src="x" onerror="alert(document.cookie)">
```

**Por qué funciona:** el navegador intenta cargar la imagen desde la URL `x`, que no existe como recurso válido. Al fallar la carga, dispara automáticamente el evento `onerror`, ejecutando el JavaScript que contiene — sin que la víctima haga clic en nada ni exista una sola etiqueta `<script>` en el payload.

### Payload 2 — SVG con `onload`

```html
<svg onload="alert(document.cookie)"></svg>
```

**Por qué funciona:** apenas el navegador renderiza el elemento `<svg>`, dispara el evento `onload` de forma automática (de manera análoga a como lo haría un `<body onload="...">` al terminar de cargar la página), ejecutando el JavaScript embebido en el atributo. Es uno de los vectores de XSS sin `<script>` más usados en la práctica, precisamente porque no requiere ninguna acción del usuario.

### (Bonus) Payload 3 — Link con esquema `javascript:`

```html
<a href="javascript:alert(document.cookie)">Click acá</a>
```

**Por qué funciona:** si la víctima hace clic en el link, el navegador ejecuta el contenido del esquema `javascript:` directamente, como si fuera código embebido. Requiere interacción del usuario (a diferencia de los dos anteriores), pero sigue sin usar la etiqueta `<script>` en ningún momento.

## Conclusión

Un filtro que solo elimina `<script>`/`</script>` ataca un caso particular, no la vulnerabilidad real. Mientras la aplicación siga insertando entrada del usuario directamente en el HTML (o en atributos HTML) sin sanitizarla ni codificarla correctamente, el atacante tiene a su disposición una superficie enorme de etiquetas y atributos capaces de ejecutar JavaScript. La defensa correcta no es una lista negra de palabras prohibidas, sino **escapar/codificar correctamente la salida** (encoding de HTML entities, Content Security Policy, uso de APIs seguras del DOM como `textContent` en vez de `innerHTML`, etc.), de forma que la entrada del usuario nunca sea interpretada como marcado o código ejecutable, sin importar qué etiqueta o atributo use.
