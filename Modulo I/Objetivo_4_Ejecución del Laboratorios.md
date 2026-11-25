# Objetivo 4: Ejecución de Laboratorios

Informe de resolución de los laboratorios

---

## Laboratorio: Votación

### Paso 1 — Identificar el endpoint de votación

Con Burp Suite como proxy, se interceptó el tráfico generado al emitir un voto desde la interfaz. Se identificó la siguiente petición:

```
POST /src/ctl/votacion.ctl.php HTTP/2
Host: chl-6ecfdc62-8f0d-457d-a13e-3c0dcd8856e7-votacion.softwareseguro.com.ar
Cookie: PHPSESSID=...; cf_clearance=...; voto=s8fvks7dk3ncq0
Content-Type: application/x-www-form-urlencoded; charset=UTF-8

opUniversidad=1
```

Donde `opUniversidad=1` corresponde al voto por la **UTN** (Harvard sería `0`).

### Paso 2 — Identificar el control anti-abuso y su falla

Al reenviar la petición original tal cual (con la cookie `voto` incluida) el servidor respondía:

```json
{"error":"No se puede votar mas de una vez", "progressbar_utn": "...457...", "progressbar_harvard": "...3521..."}
```

Se probó, en **Burp Repeater**, eliminar manualmente del request la cookie **`voto`** (dejando intactas `PHPSESSID` y `cf_clearance`) y reenviar la misma petición. El servidor **aceptó el voto nuevamente**, incrementando el contador de UTN a 458, sin devolver ningún error.

**Conclusión de la falla:** el control de "un voto por persona" está implementado únicamente a través de una cookie (`voto`) que el propio cliente controla. Al no validar la identidad del votante por ningún otro medio server-side robusto (sesión de usuario autenticado, IP, etc.), alcanza con omitir esa cookie en cada petición para que el servidor la trate como un votante nuevo — una falla de **Broken Access Control / validación de estado insuficiente del lado del servidor**.

### Paso 3 — Automatizar la explotación con Burp Intruder

Dado que la diferencia de votos a revertir era significativa (UTN 457/458 vs Harvard 3521, sobre un total de ~3978), se automatizó el envío de la petición (sin la cookie `voto`) usando **Intruder**:

1. Se envió la petición (ya sin la cookie `voto`) a Intruder.
2. En **Positions**, se limpiaron los marcadores automáticos y se marcó únicamente un valor inocuo y sin efecto en la lógica de la aplicación (el número del header `Priority: u=§1§, i`), solo para poder controlar la cantidad de repeticiones sin alterar el voto real (`opUniversidad=1` permaneció fijo en cada petición).
3. En **Payloads**, se configuró el tipo **Numbers**, secuencial, de 1 a 4000, generando así 4000 peticiones idénticas en cuanto al voto emitido.
4. Se inició el ataque (**Start attack**).

### Paso 4 — Verificación del progreso

Se verificó, inspeccionando la Response de distintas peticiones del ataque, que el contador `progressbar_utn` subía de forma consistente en cada petición exitosa (ej: de 457 a 472 en las primeras 13 peticiones), confirmando que cada petición del ataque contaba efectivamente como un voto nuevo.

### Paso 5 — Obtención del hash

Una vez que el contador de la UTN superó al de Harvard, la propia interfaz de votación mostró el mensaje de éxito con el código del desafío:

```
Desafío superado. Código: sdf7fdj843if3jhg3
```

Este código se envió como flag en la plataforma, completando el desafío.

### Resumen técnico

| | |
|---|---|
| **Tipo de vulnerabilidad** | Broken Access Control (control de "un solo voto" evadible del lado del cliente) + falta de rate limiting |
| **Herramienta usada** | Burp Suite (Repeater + Intruder) |
| **Técnica** | Eliminación de cookie de control + automatización de peticiones repetidas |

---

## Laboratorio: Apagar la IA

### Paso 1 — Reconocimiento inicial

La página principal del sitio (`/`) muestra una lista con solo 2 links, cada uno apuntando a una carpeta cuyo nombre es una cadena de 32 caracteres hexadecimales (con apariencia de hash MD5):

```
/codes/0e1422ea79781ee046484893ce0010c4/
/codes/0602940f23884f782058efac46f64b0f/
```

### Paso 2 — Contenido de los 2 archivos accesibles

Cada una de las 2 carpetas (`/codes/<hash>/`) muestra una página titulada "Mis reportes", con una lista de 25 números de entre 7 y 9 dígitos cada uno. Se descartó que el código de 16 dígitos estuviera en alguno de estos 2 archivos (ningún número individual tiene 16 dígitos), confirmando lo que indicaba el enunciado.

Tampoco se encontró directory listing habilitado al navegar directamente a `/codes/` (sin un hash específico): el servidor devuelve un error 404 "Not Found".

### Paso 3 — Identificación de la vulnerabilidad: IDOR por hash predecible

El nombre de cada carpeta no es un identificador aleatorio ni secreto: es el **hash MD5 del ID numérico del reporte, en texto plano**. Se confirmó calculando localmente el MD5 de distintos números hasta encontrar una coincidencia exacta con los 2 hashes conocidos:

```
MD5("9912") = 0e1422ea79781ee046484893ce0010c4
MD5("9995") = 0602940f23884f782058efac46f64b0f
```

**Conclusión de la falla:** se trata de un **IDOR (Insecure Direct Object Reference)** — el servidor no valida en ningún momento que el usuario tenga permiso para ver el reporte de un ID determinado; solo exige conocer el hash de ese ID. Como el hash se genera con un algoritmo público (MD5) sin ningún salt ni secreto de por medio, cualquiera puede calcular el hash de **cualquier otro ID** y acceder al reporte correspondiente a otro usuario, sin autenticarse como esa persona.

### Paso 4 — Explotación: enumeración de reportes vecinos con Burp Intruder

Dado que los 2 IDs conocidos (9912 y 9995) están muy cerca entre sí, se generó localmente una lista de hashes MD5 correspondientes a un rango de IDs cercano (9000 a 11000, 2001 valores) y se automatizó la búsqueda del reporte que contuviera el código de 16 dígitos:

1. Se envió a **Intruder** una petición `GET /codes/<hash>/` ya conocida.
2. En **Positions**, se marcó únicamente el hash de la URL como posición del payload (`GET /codes/§0602940f23884f782058efac46f64b0f§/`).
3. En **Payloads**, tipo **Simple list**, se cargó la lista de 2001 hashes MD5 precalculados (uno por cada ID del rango 9000-11000).
4. En **Settings → Grep - Match**, se configuró una expresión regular en modo Regex para que Burp marque automáticamente cualquier respuesta que contenga un `<li>` con una secuencia de 16 dígitos consecutivos — es decir, un posible código de apagado. La expresión correcta para esto es `(<li>\d{16}</li>)`, acotada al elemento de la lista (una expresión genérica `\d{16}` sin esa acotación puede dar falsos positivos si coincide con otros valores numéricos largos presentes en los headers de la respuesta).
5. Se inició el ataque (**Start attack**).

**Nota sobre el rango utilizado:** el primer intento se acotó al rango 9000-11000 (2001 valores), estimado a partir de la cercanía entre los 2 IDs conocidos (9912 y 9995). Ese rango resultó insuficiente — el ID real del reporte con el código de apagado estaba más arriba. Fue necesario ampliar el rango hasta aproximadamente 12000 (3002 valores) para dar con el reporte correcto.

### Paso 5 — Código encontrado y obtención del hash final

Al ampliar el rango y corregir el Grep-Match a la expresión acotada al `<li>`, el ataque marcó una única fila coincidente:

```
GET /codes/cdf49f5251e7b3eb4f009483121e9b64/ HTTP/2   → Status 200
Grep Match: <li>5524663362514956</li>
```

El reporte devuelto contenía, entre los números habituales de 7-9 dígitos, un valor de **16 dígitos**: `5524663362514956` — el código de apagado buscado.

Se calculó su hash MD5:

```
MD5("5524663362514956") = a8e0e8ff02dde0f62fdf4de5142d7de0
```

Este hash se envió como flag en la plataforma y fue **aceptado**, completando el desafío.

### Resumen técnico

| | |
|---|---|
| **Tipo de vulnerabilidad** | IDOR (Insecure Direct Object Reference) — identificador de recurso predecible (MD5 de un ID secuencial, sin control de autorización) |
| **Herramienta usada** | Burp Suite (vista de código fuente + Intruder con Grep Match) |
| **Técnica** | Reconstrucción del esquema de generación del identificador (MD5 de un entero) + enumeración automatizada de recursos vecinos |

---

## Laboratorio: El mejor secreto

### Paso 1 — Análisis técnico del archivo zip

Inspeccionando la estructura del zip a nivel de bytes (local file header + extra field + directorio central), se identificó que el archivo está cifrado con **WinZip AE-1 / AES-256**: salt de 16 bytes, verificador de contraseña de 2 bytes, texto cifrado, y un MAC (HMAC-SHA1 truncado) de 10 bytes al final. La derivación de claves usa PBKDF2-HMAC-SHA1 con 1000 iteraciones.

Se implementó el descifrado completo en Python (derivación de claves + verificación de HMAC + descifrado AES-256-CTR + descompresión DEFLATE) y se **verificó su correctitud** cifrando y descifrando un archivo de prueba propio con contraseña conocida, confirmando que la implementación es correcta byte a byte contra el formato real del zip entregado.

### Paso 2 — Análisis del video: extracción de la secuencia de botones

Se analizó el video cuadro por cuadro (zoom sobre el teclado, segmentación por energía de audio para detectar los instantes de pulsación) para determinar qué botón se presiona en cada momento. Se detectaron **12 pulsaciones** en total, sobre un teclado con apariencia inusual (varios botones distintos pueden mostrar/sonar como el mismo dígito).

Secuencia de botones identificada (etiquetando cada botón distinto con una letra, en el orden cronológico en que se presionan):

```
A B C C D A D D D D E C
```

Al identificar visualmente el dígito real de cada botón, la secuencia quedó:

```
6 2 6 6 9 6 9 9 9 9 5 6
```

(Dos botones físicamente distintos —A y C— corresponden ambos al dígito 6 en este teclado.)

### Paso 3 — Intentos de ataque por fuerza bruta

Se probaron varias hipótesis sobre cómo se construye la contraseña a partir de la secuencia observada, descifrando localmente contra el archivo real en cada caso:

1. **Permutación de 10 dígitos únicos (0-9) sin repetir** (hipótesis inicial, antes de tener la pista de la secuencia): 3.628.800 combinaciones probadas — sin éxito.
2. **Secuencia literal** `626696999956` como contraseña — sin éxito.
3. **Todas las permutaciones únicas del multiconjunto** de 12 dígitos observado (cinco 6, cinco 9, un 2, un 5): 33.264 combinaciones — sin éxito.
4. **Patrón fijo con asignación de dígito por letra** (manteniendo el orden `A B C C D A D D D D E C`, probando qué dígito real corresponde a cada una de las 5 letras distintas): 30.240 combinaciones con los 5 dígitos distintos entre sí, y 100.000 combinaciones permitiendo dígitos repetidos entre letras — sin éxito en ninguno de los dos casos.

En todos los casos se descartaron falsos positivos del verificador de 2 bytes (estadísticamente esperables) comprobando también el HMAC completo y el descifrado real.

### Conclusión

No se logró encontrar la contraseña correcta del archivo `secreto1.zip`. Se descifró correctamente la lógica general del desafío (una secuencia de pulsaciones en un teclado que de algún modo determina la contraseña) y se extrajo una secuencia concreta a partir del análisis del video, pero ninguna de las transformaciones de esa secuencia probadas (permutación directa, patrón con asignación de dígitos) resultó ser la contraseña válida. Cabe destacar que este fue, según indicó el profesor, el único de los desafíos que tampoco él logró resolver completamente — solo brindó la pista de la secuencia de 12 pulsaciones como punto de partida, sin solución confirmada.

### Resumen técnico

| | |
|---|---|
| **Tipo de vulnerabilidad / técnica** | Fuerza bruta sobre contraseña de archivo ZIP cifrado (AES-256, formato WinZip AE-1), guiada por una secuencia de pulsaciones extraída de un video |
| **Herramienta usada** | Análisis manual de video/audio + script propio en Python (PBKDF2-HMAC-SHA1 + AES-256-CTR + HMAC-SHA1 + DEFLATE) |
| **Resultado** | No resuelto |

---

## Laboratorio: Votación nueva versión

### Paso 1 — Identificar el endpoint de votación

Con Burp Suite, se interceptó la petición generada al emitir un voto. A diferencia de la versión anterior del laboratorio, el endpoint no aparece referenciado directamente en el HTML de la página principal (`/`): el manejo del voto está en un archivo JavaScript externo, `js/index.tmp.js`, localizado buscando en el código fuente las etiquetas `<script src="...">`. Dentro de ese archivo se encontró el handler del botón "Votar":

```js
$('#btnVotar').click(function(){
    var op = $("input[name='universidad']:checked").val();
    var data = { "opUniversidad": op };
    $.ajax({
        type: "POST",
        url: "src/ctl/votacion.ctl.php",
        data: data,
        dataType: "json",
        success: function(respuesta){
            if (respuesta['error'] != undefined) {
                $("#divMensaje").html("<p class='error'>"+respuesta['error']+"</p>");
            } else {
                $("#divProgressBarUTN").html(respuesta['progressbar_utn']);
                $("#divProgressBarHarvard").html(respuesta['progressbar_harvard']);
            }
        }
    });
});
```

La petición real observada en Burp:

```
POST /src/ctl/votacion.ctl.php HTTP/2
Host: chl-ba8a8c77-bb6b-4da7-a28b-d644145bf0a7-votacion-nueva-version.softwareseguro.com.ar
Cookie: PHPSESSID=...
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
X-Requested-With: XMLHttpRequest

opUniversidad=1
```

Donde `opUniversidad=1` es un voto por la **UTN** (Harvard sería `0`).

### Paso 2 — Identificar el nuevo control anti-abuso y su falla

A diferencia de la versión original (que usaba una cookie `voto` controlable por el cliente), esta versión reforzó la defensa: el servidor ahora valida **una sola votación por dirección IP**. Al reenviar la petición de voto desde la misma conexión una segunda vez, la respuesta es:

```json
{"error":"No se puede votar mas de una vez desde la misma ip","progressbar_utn":"...","progressbar_harvard":"..."}
```

Se probó agregar manualmente, en **Burp Repeater**, el header `X-Forwarded-For` con una IP arbitraria no utilizada antes:

```
X-Forwarded-For: 203.0.113.55
```

El servidor **aceptó el voto nuevamente**, incrementando el contador de la UTN, pese a provenir de la misma conexión/sesión real.

**Conclusión de la falla:** el control de "un voto por IP" confía ciegamente en el header `X-Forwarded-For`, que es **enviado y controlado por el propio cliente**. El servidor lo usa como si fuera la IP real del visitante (en lugar de obtenerla de la conexión TCP/del proxy de confianza), sin validar su autenticidad. Esto es una falla de **Broken Access Control por confianza indebida en un header HTTP falsificable (IP Spoofing vía `X-Forwarded-For`)**: alcanza con cambiar ese valor en cada petición para que el servidor trate cada voto como proveniente de un visitante distinto.

### Paso 3 — Automatizar la explotación con Burp Intruder

Antes de automatizar, con el estado inicial UTN=1502 / Harvard=1602 (Harvard ganando por 100 votos sobre un total de ~3104), se configuró un ataque en **Intruder**:

1. Se envió a Intruder la petición de voto (`opUniversidad=1`) con el header `X-Forwarded-For: 203.0.113.55` ya agregado.
2. En **Positions**, se marcó únicamente el último octeto de la IP falsa como posición del payload: `X-Forwarded-For: 203.0.113.§55§`.
3. En **Payloads**, se configuró el tipo **Numbers**, secuencial, de 1 a 254 (paso 1), generando 254 peticiones con una IP falsa distinta en cada una (`203.0.113.1` a `203.0.113.254`), todas votando por la UTN.
4. Se inició el ataque (**Start attack**).

### Paso 4 — Verificación del progreso y confirmación de la condición

Mientras el ataque corría, se verificaron en **Burp Repeater** (reenviando el voto con IPs frescas fuera del rango 1-254, ej. `203.0.114.1`) los valores devueltos en el JSON de respuesta:

```json
{"progressbar_utn":"...1660...","progressbar_harvard":"...1602..."}
```

Confirmando que, tras la automatización, **la cantidad de votos de la UTN superó a la de Harvard** (estado final verificado: UTN ≈ 1722 vs Harvard = 1602, Harvard dejó de recibir nuevos votos porque el ataque solo emite votos por la UTN).

### Paso 5 — Obtención del hash

Se inspeccionó exhaustivamente la respuesta del endpoint `votacion.ctl.php` (incluso una vez cumplida la condición UTN > Harvard) y el archivo `js/index.tmp.js`, sin encontrar que el propio endpoint devuelva un código o hash adicional en ningún momento: solo actualiza los dos contadores (`progressbar_utn` y `progressbar_harvard`). Esto indica que la obtención del código HASH prometido por el enunciado se gestiona desde la plataforma del curso (Software Seguro) una vez que detecta la condición cumplida, y no desde la propia aplicación vulnerable.

Una vez cumplida la condición UTN > Harvard, la plataforma del curso (Software Seguro) desbloqueó el código HASH correspondiente al desafío:

```
143885b3abc1012375b3846f84c39203
```

Este código se envió como flag en la plataforma, confirmando la resolución exitosa del laboratorio.

### Resumen técnico

| | |
|---|---|
| **Tipo de vulnerabilidad** | Broken Access Control — confianza indebida en el header `X-Forwarded-For` (spoofing de IP) para el control de "un voto por IP" |
| **Herramienta usada** | Burp Suite (búsqueda en código fuente + Repeater + Intruder) |
| **Técnica** | Identificación del endpoint vía JS externo + falsificación de IP de origen vía header + automatización de 254 peticiones con IPs distintas |
