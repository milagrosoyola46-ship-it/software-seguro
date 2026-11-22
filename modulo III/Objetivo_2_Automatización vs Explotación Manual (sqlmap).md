# Objetivo 2: Automatización vs Explotación Manual (sqlmap)

## Por qué sqlmap no es "la solución universal"

sqlmap es extremadamente potente porque automatiza un proceso que, hecho a mano, sería larguísimo: detectar el tipo de inyección, identificar el motor de base de datos, extraer tablas y columnas, y volcar datos — todo con un solo comando. Pero sqlmap funciona bajo un **modelo de request/response relativamente simple y predecible**: toma una petición HTTP (o una URL), identifica un parámetro, y prueba sistemáticamente una batería enorme de payloads conocidos contra ese punto de inyección, uno a la vez, en una secuencia más o menos rígida de peticiones.

Ese mismo modelo —genérico, basado en patrones conocidos, con un volumen alto de peticiones repetitivas— es exactamente lo que falla cuando el escenario real se aleja de un caso simple de "un parámetro, una petición, sin contexto adicional".

## Escenario 1: WAF / sistema anti-automatización con detección de patrones y volumen

**Por qué falla sqlmap:** sqlmap, en su configuración por defecto, genera un volumen alto de peticiones (cientos o miles) con payloads que siguen patrones muy reconocibles (`' OR 1=1--`, `SLEEP(5)`, `UNION SELECT NULL,NULL,...`, headers de User-Agent característicos de la herramienta si no se los cambia explícitamente). Un WAF moderno (Cloudflare, AWS WAF, ModSecurity con reglas OWASP CRC) o un sistema de rate-limiting/anti-bot puede:
- Reconocer la firma de esos payloads y bloquear la petición específica.
- Detectar el **volumen y la cadencia anómala** de peticiones casi idénticas en poco tiempo, y banear la IP o la sesión completa, cortando el ataque antes de que sqlmap llegue a confirmar la inyección.

**Por qué un atacante manual sí lo logra:** un pentester experimentado, trabajando directamente en **Burp Repeater**, puede:
- Enviar **una cantidad mínima y espaciada de peticiones** (una por vez, con pausas deliberadas), en vez de la ráfaga automática de sqlmap.
- **Ofuscar manualmente el payload** de formas creativas que un WAF genérico no tiene en su lista de firmas (ej. usar comentarios inline `/**/` entre palabras clave SQL, alternar mayúsculas/minúsculas, usar codificaciones alternativas de caracteres, concatenación de strings para evitar la palabra `UNION` literal), ajustando el payload iterativamente según la respuesta exacta que ve en cada intento — algo que sqlmap no puede "improvisar" sin que el humano le indique explícitamente qué tamper script usar (y muchas veces ni los tampers incluidos alcanzan para una regla de WAF específica y custom de esa empresa).
- Detenerse apenas confirma la inyección con un solo payload exitoso, sin necesitar la batería completa de pruebas que sqlmap ejecuta por defecto para caracterizar el motor y la técnica.

## Escenario 2: Inyección SQL de segundo orden (Second-Order SQLi)

**Por qué falla sqlmap:** una inyección de segundo orden ocurre cuando el payload malicioso se **guarda** en un primer request (por ejemplo, al registrar un usuario con un nombre como `admin'--`), pero la inyección **no se ejecuta en ese mismo momento** — se ejecuta más tarde, en un endpoint completamente distinto, cuando ese dato guardado es usado dentro de otra consulta SQL (por ejemplo, cuando un panel de administración construye un reporte que incluye el nombre de ese usuario en una consulta sin parametrizar).

sqlmap trabaja, por diseño, sobre **una sola petición a la vez**: envía un payload y analiza la respuesta de **esa misma petición** para determinar si hubo inyección. No tiene forma automática de entender que el efecto de su payload va a aparecer recién en una petición distinta, a un endpoint distinto, posiblemente minutos u horas después, ni de relacionar causalmente ambos eventos sin ayuda manual explícita del operador.

**Por qué un atacante manual sí lo logra:** un pentester que entiende el flujo de la aplicación puede:
1. Identificar manualmente, revisando la lógica de negocio de la app (no solo probando parámetros al azar), **dónde se guarda un dato** y **dónde se vuelve a usar** ese mismo dato más adelante en otra consulta (por ejemplo, intuyendo que el nombre de usuario seguramente aparece después en logs de auditoría, reportes de administrador, o notificaciones por email generadas por un proceso en segundo plano).
2. Insertar el payload malicioso en el primer punto (el registro), y luego **navegar manualmente hasta el segundo punto** (el panel de admin, el reporte, etc.) para verificar el efecto, correlacionando ambos pasos con el contexto completo de la aplicación que solo un humano auditando el flujo completo puede reconstruir.

## (Opcional) Escenario 3: Inyección dentro de un flujo multi-paso con tokens dinámicos

**Por qué falla sqlmap:** si el parámetro vulnerable está embebido en un flujo donde cada petición requiere un token anti-CSRF o un nonce que cambia en cada carga de página (y que se invalida si se reutiliza), el modelo de sqlmap de "repetir la misma petición cientos de veces cambiando solo el payload" se rompe: cada repetición con un token ya usado es rechazada por el servidor antes siquiera de llegar a evaluar el SQL, generando falsos negativos (sqlmap concluye "no es vulnerable" cuando en realidad nunca llegó a probar el payload contra la base de datos).

**Por qué un atacante manual sí lo logra:** usando Burp Suite, un pentester puede configurar reglas de manejo de sesión (*session handling rules*) o simplemente, de forma manual en Repeater, **obtener un token fresco antes de cada intento** y ajustar el payload en la misma petición donde ese token todavía es válido — algo que requiere entender y reproducir manualmente la lógica del flujo, en vez de simplemente repetir una petición grabada.

## Conclusión

Depender exclusivamente de sqlmap asume un escenario "ideal" donde el punto de inyección es simple, aislado, estático en el tiempo y sin defensas activas monitoreando el tráfico — condiciones que rara vez se dan por completo en una aplicación real bien protegida. Un pentester experimentado no reemplaza a sqlmap, lo complementa: usa su comprensión del flujo de negocio, de las defensas específicas del objetivo, y su capacidad de ajustar manualmente cada petición en Burp Suite para los casos donde el modelo genérico y automatizado de la herramienta simplemente no tiene forma de adaptarse.
