# Objetivo 3: Anatomía de una Condición de Carrera (Race Condition)

## TOCTOU (Time-Of-Check to Time-Of-Use)

Una condición de carrera en seguridad web ocurre porque casi toda operación "segura" en apariencia en realidad se ejecuta en **dos pasos separados en el tiempo**, no como una sola operación atómica:

1. **Time-Of-Check (verificación):** el servidor consulta el estado actual — por ejemplo, "¿este código de descuento todavía no fue usado?".
2. **Time-Of-Use (uso/acción):** el servidor ejecuta la acción basada en esa verificación — por ejemplo, "aplicar el descuento y marcar el código como usado".

Entre el paso 1 y el paso 2 existe una **ventana de tiempo**, por más pequeña que sea (milisegundos), durante la cual el estado todavía no fue actualizado. Si el servidor recibe **otra petición idéntica durante esa ventana**, esa segunda petición hace su propia verificación en el paso 1 — y como el primer hilo todavía no llegó al paso 2 (todavía no marcó el código como usado), la verificación le devuelve "sí, el código es válido" otra vez, aunque en realidad ya está siendo procesado por otra petición en paralelo.

## Cómo un atacante fuerza esta ventana con Turbo Intruder

En condiciones normales (un usuario común, haciendo click una vez), esta ventana de milisegundos nunca se nota, porque las peticiones llegan de a una, separadas en el tiempo por la velocidad humana de interacción — el paso 2 de la primera petición siempre termina mucho antes de que llegue una segunda. El atacante necesita entonces **forzar artificialmente** que muchas peticiones lleguen al servidor en el instante exacto en que esa ventana está abierta, maximizando la probabilidad de que varias caigan dentro de ella.

Acá es donde entra **Turbo Intruder** (una extensión de Burp Suite pensada específicamente para esto, a diferencia del Intruder clásico que envía peticiones de forma más espaciada): permite enviar una ráfaga de peticiones **casi simultáneas**, abriendo muchas conexiones en paralelo y minimizando el tiempo entre el envío de cada una, en vez de esperar la respuesta de una antes de mandar la siguiente (que es como funciona una navegación normal o incluso el Intruder clásico en su modo por defecto).

### El flujo del ataque:

1. El atacante identifica un endpoint de canje de código de descuento (por ejemplo, `POST /api/cupon/canjear` con `{"codigo": "DESCUENTO50"}`), confirmando primero con una petición normal que el código es válido y de **un solo uso**.
2. Usando Turbo Intruder, el atacante configura un script que envía, por ejemplo, **50 copias idénticas** de esa misma petición de canje, todas con el mismo código, disparadas en el menor tiempo posible entre sí (Turbo Intruder usa su propio motor HTTP de bajo nivel para lograr esto, evitando los cuellos de botella de conexión que tendría una herramienta más genérica).
3. El servidor recibe las 50 peticiones casi al mismo tiempo. Si la lógica del backend está implementada como:
   ```
   SELECT usado FROM cupones WHERE codigo = 'DESCUENTO50'   -- (verificación)
   -- ... aplica el descuento al carrito ...
   UPDATE cupones SET usado = true WHERE codigo = 'DESCUENTO50'  -- (actualización, recién acá)
   ```
   Varios de esos 50 hilos/procesos en paralelo van a ejecutar el `SELECT` (la verificación) **antes** de que cualquiera de ellos llegue a ejecutar el `UPDATE` (la actualización) — porque el servidor está procesando varias peticiones a la vez, y la verificación de cada una no espera a que las demás terminen su ciclo completo.
4. Como resultado, varias (o todas) esas 50 peticiones ven `usado = false` en su verificación, y **todas** proceden a aplicar el descuento, aunque el código estaba pensado para usarse una sola vez. El atacante termina canjeando el cupón de un solo uso múltiples veces — en un sistema financiero, esto podría significar transferir o descontar dinero 50 veces en vez de una.

## Por qué esto es particularmente crítico en sistemas financieros o de inventario

En estos sistemas, el "check" suele ser exactamente del tipo "¿hay suficiente saldo/stock para esta operación?", y el "use" es "descontar ese saldo/stock". Si un atacante logra disparar, por ejemplo, 50 peticiones simultáneas de retiro de \$100 contra una cuenta que solo tiene \$100 de saldo, y la verificación de saldo de cada una de esas 50 peticiones ocurre **antes** de que cualquiera de ellas descuente el saldo real, el sistema podría terminar procesando los 50 retiros (\$5000 en total) sobre una cuenta que nunca debería haber permitido más de uno.

## Cómo se previene esto en el backend

La solución no es "procesar las peticiones más rápido" ni filtrarlas por velocidad (eso sería tratar el síntoma, no la causa) — la solución real es garantizar que el "check" y el "use" ocurran como **una sola operación atómica e indivisible**, de forma que ninguna otra petición pueda "colarse" en el medio:

- **Bloqueos a nivel de fila en la base de datos** (`SELECT ... FOR UPDATE` en PostgreSQL/MySQL), que bloquean esa fila específica hasta que la transacción actual termine, obligando a cualquier otra petición concurrente para el mismo recurso a esperar su turno.
- **Transacciones con nivel de aislamiento `Serializable`**, el nivel más estricto, que garantiza que el resultado final sea equivalente a si todas las transacciones se hubieran ejecutado una después de la otra, nunca en paralelo sobre el mismo dato.
- **Operaciones atómicas a nivel de aplicación** (locks/mutex) cuando el estado no vive únicamente en la base de datos, asegurando que solo un hilo del servidor pueda entrar a la sección crítica (verificación + actualización) para un mismo recurso a la vez.
- **Restricciones a nivel de base de datos** que haga la operación fallar de forma segura ante un intento duplicado (por ejemplo, una actualización condicional como `UPDATE cupones SET usado = true WHERE codigo = 'X' AND usado = false`, verificando que la fila afectada sea exactamente 1 — si da 0 filas afectadas, significa que otra petición ya lo marcó como usado primero, y esa segunda petición debe fallar).

## Conclusión

Una condición de carrera explota el hecho de que "verificar" y "actuar" casi nunca son la misma operación atómica en el código de una aplicación típica, y herramientas como Turbo Intruder existen específicamente para maximizar la probabilidad de que varias peticiones caigan dentro de esa ventana de tiempo, forzando que una acción pensada para ejecutarse una sola vez se ejecute varias veces en paralelo. La única mitigación real es eliminar esa ventana a nivel de diseño, garantizando atomicidad mediante bloqueos de base de datos o transacciones con aislamiento estricto, nunca confiando en que "la aplicación es rápida" como forma de evitarlo.
