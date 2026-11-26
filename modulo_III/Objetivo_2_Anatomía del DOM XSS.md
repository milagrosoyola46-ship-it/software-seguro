# Objetivo 2: Anatomía del DOM XSS

A diferencia del XSS Almacenado (Stored) y el Reflejado (Reflected), el **DOM-Based XSS** es indetectable para muchos sistemas de prevención de intrusos (IPS) y firewalls de aplicación web (WAF). ¿Por qué ocurre esto?

## Primero: dónde ocurre cada tipo de XSS

Para entender por qué el DOM XSS es especial, hay que ubicar dónde se procesa el payload malicioso en cada caso:

| Tipo | ¿Dónde se inyecta/refleja el payload? | ¿Pasa por el servidor? |
|---|---|---|
| **Stored XSS** | El payload se guarda en una base de datos del servidor y se devuelve en el HTML de respuestas futuras | Sí, dos veces: al guardarlo y al devolverlo |
| **Reflected XSS** | El servidor toma un valor de la petición (ej. un parámetro de query) y lo refleja directamente en el HTML de la respuesta | Sí, el servidor arma la respuesta incluyendo el payload |
| **DOM-Based XSS** | El payload **nunca llega al servidor** — un script JavaScript del lado del cliente lee datos controlables por el atacante y los inserta de forma insegura en el DOM de la página, todo dentro del navegador | **No**, el servidor nunca ve ese dato |

Esa última fila es la clave de todo: en el DOM XSS, la totalidad del ataque ocurre **en el navegador de la víctima**, sin que el valor malicioso viaje nunca en la petición HTTP hacia el servidor.

## Por qué los IPS y WAF no lo detectan

Un WAF o un IPS funcionan analizando el **tráfico de red**: inspeccionan las peticiones HTTP entrantes (parámetros GET/POST, headers, body) y las respuestas del servidor, buscando patrones sospechosos (como `<script>`, `onerror=`, etc.) en esos datos que efectivamente circulan por la red.

El problema es justamente que, en DOM XSS, **el payload puede no viajar nunca dentro de la petición HTTP que el servidor (y por lo tanto el WAF, que está delante del servidor) llega a inspeccionar**. El WAF solo puede analizar lo que pasa por la red; no tiene visibilidad de lo que el JavaScript hace *dentro* del navegador de la víctima, después de que la página ya fue cargada y entregada.

### El rol del fragmento (`#`) en la URL

La pista más concreta de esto es el **fragmento de URL** (todo lo que va después del símbolo `#`), también llamado "hash" de la URL. Por especificación del protocolo HTTP, **el fragmento de una URL nunca se envía al servidor**. Por ejemplo, ante esta URL:

```
https://ejemplo.com/pagina?buscar=algo#<img src=x onerror=alert(1)>
```

El navegador, al hacer la petición HTTP real, solo envía:

```
GET /pagina?buscar=algo HTTP/1.1
Host: ejemplo.com
```

La parte `#<img src=x onerror=alert(1)>` **se queda únicamente del lado del cliente**, disponible para JavaScript a través de `window.location.hash` (o `location.hash`), pero **jamás llega al servidor, ni a ningún proxy o WAF que esté inspeccionando el tráfico HTTP**. Es, literalmente, invisible para cualquier dispositivo de red: el fragmento ni siquiera figura en los logs de acceso del servidor web.

### Qué pasa si el JavaScript de la página procesa `location.hash` de forma insegura

Si el código JavaScript de la página hace algo como:

```javascript
// Código vulnerable
var mensaje = location.hash.substring(1); // saca el '#'
document.getElementById("resultado").innerHTML = decodeURIComponent(mensaje);
```

Este script toma el valor del fragmento (que el atacante controla completamente, induciendo a la víctima a hacer clic en un link con un `#` armado) y lo inserta directamente en el DOM vía `innerHTML`, sin sanitizarlo. El navegador interpreta ese contenido como HTML real, y si contiene algo como `<img src=x onerror=alert(1)>`, lo ejecuta.

Todo este flujo —lectura del hash, inserción insegura en el DOM, ejecución del payload— ocurre **enteramente dentro del navegador**, usando una API (`location.hash`) cuyo valor **nunca pasa por el servidor ni por ningún dispositivo de inspección de tráfico de red**.

## Por qué esto hace indetectable al DOM XSS para WAF/IPS

Juntando las piezas:

1. El WAF/IPS solo puede inspeccionar lo que viaja en la red (peticiones y respuestas HTTP).
2. El fragmento de la URL, por diseño del protocolo, **no se envía nunca al servidor** — se queda en el navegador.
3. Si la vulnerabilidad surge de que un script del lado del cliente procesa ese fragmento (u otras fuentes puramente del lado del cliente, como `document.referrer`, `window.name`, o datos ya presentes en el DOM) de forma insegura, **no existe ningún paquete de red que el WAF pueda inspeccionar y en el que encuentre el payload**, porque el payload nunca viajó por la red en primer lugar.

En otras palabras: no es que el WAF "falle" en detectar el patrón — es que **estructuralmente no tiene acceso al dato** que origina el ataque, porque ese dato nunca formó parte del tráfico HTTP que el WAF está en condiciones de observar. Por este motivo, la defensa contra DOM XSS no puede depender de dispositivos de red: tiene que resolverse **en el código JavaScript del lado del cliente**, evitando el uso inseguro de fuentes controlables por el atacante (`location.hash`, `location.search`, `document.URL`, etc.) con sumideros peligrosos (`innerHTML`, `document.write`, `eval`), y reemplazándolos por APIs seguras (`textContent`, `createElement` + `setAttribute`, frameworks que escapan automáticamente la salida, etc.).
