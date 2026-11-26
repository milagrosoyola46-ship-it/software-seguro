# Objetivo 5: El Espejismo del Cifrado en el Cliente (Frontend Encryption)

## Escenario

Una aplicación financiera cifra, mediante JavaScript ejecutado en el navegador del usuario, el body completo de las peticiones POST y los parámetros sensibles, antes de enviarlos al backend. Los arquitectos afirman que esto hace **imposible** que un atacante intercepte y manipule los valores con un proxy como Burp Suite, ya que "todo viaja cifrado".

Esta afirmación es una **falsa sensación de seguridad**, y a continuación se explica por qué, dónde está el fallo de diseño, y cómo lo explotaría un atacante paso a paso.

---

## El fallo fundamental de diseño

### El secreto (clave de cifrado) tiene que viajar hasta el propio atacante

Para que el navegador pueda cifrar datos con JavaScript, **el código que implementa el cifrado —y, casi siempre, la clave o el material necesario para derivarla— tiene que entregarse al cliente como parte del propio código fuente de la página** (un archivo `.js` que el navegador descarga y ejecuta). Ese archivo JavaScript es completamente visible, legible y modificable por cualquiera que controle el navegador donde se ejecuta — que es exactamente lo que controla un atacante.

Esto es radicalmente distinto de TLS/HTTPS (el cifrado de transporte real): en TLS, el atacante que intercepta el tráfico en la red **no tiene acceso a las claves privadas** del servidor ni del certificado — esas claves nunca abandonan servidores confiables. En el "cifrado en el cliente" descripto en este escenario, en cambio, **el atacante es, literalmente, el mismo entorno de ejecución que tiene la clave y el algoritmo completos a su disposición**, porque es su propio navegador el que los descargó y los ejecuta.

En criptografía esto se conoce como el problema de **"whitebox cryptography"** (criptografía de caja blanca): cualquier esquema que necesite mantener un secreto frente a un adversario que controla por completo el entorno donde ese secreto se usa, está condenado a fallar tarde o temprano — no es una cuestión de "si", sino de "cuánto esfuerzo le toma al atacante".

### Por qué cifrar en el cliente no aporta seguridad real

El cifrado solo tiene sentido quien lo controla el adversario. Acá, el "adversario" (el usuario/atacante) es exactamente quien ejecuta el cifrado. Entonces, cifrar la petición antes de que salga del navegador **no protege nada frente a ese mismo navegador** — solo protegería (de forma redundante, porque ya lo hace TLS) frente a alguien que intercepte el tráfico de red *sin* tener acceso al navegador de la víctima, que no es el escenario relevante para "manipular valores con un proxy como Burp Suite" (ahí el atacante y el dueño del navegador son la misma persona).

---

## Pasos tácticos que realizaría un atacante para evadir esta protección

### Paso 1 — Confirmar que el cifrado ocurre del lado del cliente

Interceptando el tráfico con Burp Suite (Proxy) y mirando la pestaña Network de las DevTools del navegador, el atacante nota que el body de las peticiones POST es un blob ilegible (ej. base64 de datos cifrados), pero que **la página carga uno o varios archivos `.js` propios** del sitio. Revisando el código fuente de esos archivos (Burp también los captura, o directamente con Ctrl+U / el panel "Sources" de DevTools), encuentra las funciones de cifrado (por ejemplo, usando `CryptoJS`, `forge`, o una implementación propia de AES/RSA) y, muy probablemente, **la clave o semilla embebida directamente en el código**, o un mecanismo de derivación que depende únicamente de datos también disponibles en el cliente.

### Paso 2 — Replicar o invocar directamente la función de cifrado

El atacante no necesita "romper" ningún algoritmo criptográfico. Simplemente usa la **consola de JavaScript del navegador** (DevTools) para **llamar directamente a la misma función de cifrado que usa la aplicación legítima**, pasándole como argumento los valores que él mismo quiere enviar (por ejemplo, un monto de transferencia distinto, o un ID de cuenta ajeno):

```javascript
// Ejecutado directamente en la consola del navegador de la víctima/atacante
let payloadManipulado = { monto: 999999, cuentaDestino: "CUENTA_DEL_ATACANTE" };
let cifrado = funcionDeCifradoDeLaApp(JSON.stringify(payloadManipulado), claveEncontrada);
console.log(cifrado); // esto es exactamente lo que la app "legítima" enviaría
```

Como el atacante tiene la función real, la clave real, y el algoritmo real —todo entregado por la propia aplicación—, puede generar un blob cifrado **indistinguible de uno legítimo**, pero con los valores que él eligió.

### Paso 3 — Enviar el valor manipulado con Burp Repeater

Con el blob cifrado recién generado (conteniendo los datos manipulados), el atacante intercepta una petición legítima con Burp Proxy, la envía a **Repeater**, y reemplaza el body original por el blob cifrado que generó en el paso anterior. Al servidor le llega una petición cifrada "válida" desde su perspectiva (porque fue cifrada con el algoritmo y clave correctos), pero cuyo contenido real fue completamente elegido por el atacante.

### Paso 4 (alternativa) — Interceptar el valor en texto plano con un breakpoint, sin necesidad de replicar nada

Si el atacante prefiere no analizar ni replicar el algoritmo de cifrado, puede directamente usar el **debugger de DevTools**: coloca un breakpoint justo antes de la línea donde se invoca la función de cifrado (en el archivo `.js` de la app), deja que la aplicación arme la petición normalmente desde la interfaz, y cuando la ejecución se detiene en el breakpoint:

- **Lee** el valor en texto plano que está a punto de cifrarse (resolviendo también el objetivo de "leer los datos", si el servidor también devuelve datos cifrados que se descifran en el cliente — ahí el atacante pone el breakpoint después del descifrado, en lugar de antes del cifrado).
- **Modifica en vivo**, desde el propio panel de variables de DevTools, el valor que está a punto de cifrarse (por ejemplo, cambia `monto: 100` por `monto: 999999`).
- **Reanuda la ejecución**: el propio código legítimo de la aplicación cifra el valor ya manipulado, con la clave real, y lo envía — sin que el atacante haya tenido que escribir una sola línea de criptografía.

Esta variante es, en la práctica, la más usada: no requiere entender el algoritmo en absoluto, solo pausar la ejecución del script en el momento justo y tocar la variable antes de que se cifre.

### Paso 5 — (Para herramientas como Burp Suite específicamente) Automatizar el proceso

Si el atacante quiere seguir usando Burp Intruder/Repeater de forma más cómoda para probar muchos valores distintos, puede escribir una pequeña extensión de Burp o un script externo (Python/Node) que **reimplemente la función de cifrado encontrada en el JS** (copiando el mismo algoritmo, modo y clave), de forma de poder generar automáticamente payloads cifrados válidos para cualquier valor, sin depender de la consola del navegador en cada intento.

---

## Conclusión

El fallo de diseño no está en el algoritmo de cifrado elegido (podría ser AES-256 perfectamente implementado y seguiría fallando), sino en la **premisa arquitectónica**: intentar ocultarle un secreto (la lógica y/o clave de cifrado) exactamente al mismo actor del que se lo quiere ocultar (el usuario que controla su propio navegador). Cualquier cifrado cuya clave y algoritmo deban entregarse al cliente para que el cliente mismo cifre, es trivialmente reversible por ese cliente — ya sea leyendo el código, invocando sus funciones directamente, o interceptando los valores en memoria antes de que se cifren.

La protección real contra la manipulación de parámetros no puede delegarse al cliente bajo ninguna forma: el servidor debe **validar y autorizar cada valor recibido de forma independiente** (montos, IDs de cuenta, permisos), sin asumir nunca que "si llegó cifrado desde el frontend, es confiable". El cifrado en tránsito ya lo resuelve correctamente TLS/HTTPS (que sí protege contra un atacante que *no* controla ninguno de los dos extremos de la comunicación); agregar una capa de cifrado adicional en el cliente no mejora esa protección y, peor aún, genera una falsa sensación de seguridad que puede llevar a relajar controles de validación que sí son indispensables del lado del servidor.
