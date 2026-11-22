# Objetivo 4: El Punto Ciego de los Escáneres Automatizados

## Por qué los escáneres son excelentes para fallas técnicas

Herramientas como Nessus, Acunetix o el escáner activo de Burp Suite funcionan enviando automáticamente miles de peticiones con payloads conocidos (firmas de XSS, SQLi, path traversal, etc.) y comparando la respuesta contra patrones predecibles: un mensaje de error de SQL, un script reflejado en el HTML, un header de seguridad ausente, una versión de software desactualizada. Son excelentes en esto porque **la falla y su "firma" de detección están claramente definidas de antemano** — el escáner solo necesita reconocer un patrón conocido, sin entender qué hace la aplicación ni para quién.

## Por qué la lógica de negocio es el mayor "punto ciego"

Una vulnerabilidad de lógica de negocio (como las que vimos en este módulo: saltar un paso del checkout, manipular una cantidad a negativo, explotar una condición de carrera) **no tiene ninguna firma técnica que un escáner pueda reconocer** — no rompe la sintaxis de nada, no genera un error, y la petición en sí es **perfectamente válida** desde el punto de vista del protocolo HTTP y del código del servidor.

Para detectar que forzar la navegación directa al paso 3 del checkout evita el pago, el escáner tendría que **entender el propósito y las reglas del negocio**: que existe un flujo de 3 pasos, que el paso 2 es obligatorio, y que "orden creada sin pago" es un resultado anómalo. Nada de eso es visible en la estructura de la petición o la respuesta por sí sola — un escáner ve una petición `GET /api/checkout/success` que devuelve `200 OK`, exactamente el mismo tipo de respuesta "exitosa" que vería en cualquier flujo legítimo. **No hay ningún payload malicioso que inyectar ni ningún patrón de error que buscar**, porque el atacante no rompió nada técnicamente: usó la aplicación exactamente como fue programada, solo que en un orden o con un valor que el diseño no contempló como abusivo.

En otras palabras: el escáner puede verificar **"¿esto está roto?"** (comparando contra reglas técnicas universales, válidas para cualquier aplicación), pero no puede verificar **"¿esto tiene sentido para este negocio en particular?"** — esa pregunta requiere conocer las reglas específicas e implícitas de esa aplicación en particular, que no están escritas en ningún estándar que una herramienta genérica pueda consultar.

## Por qué la auditoría manual es indispensable en esta etapa

Un humano auditando la aplicación entiende el **contexto de negocio**: sabe que un carrito de compras no debería terminar con saldo a favor, que un checkout de 3 pasos implica que el paso 2 (pago) es un requisito antes del paso 3, que un balance de cuenta nunca debería quedar negativo después de una compra. Esa comprensión del "flujo esperado" es lo que permite notar cuándo un comportamiento, aunque técnicamente válido y sin errores, **no tiene sentido** — y es exactamente lo que ningún escáner genérico puede replicar, porque cada aplicación tiene reglas de negocio distintas y propias, que no pueden preconfigurarse como una firma universal de ataque.

## Conclusión

Los escáneres automatizados son la herramienta correcta para encontrar fallas técnicas conocidas a gran escala y de forma rápida, pero son ciegos ante vulnerabilidades de lógica de negocio porque estas no rompen ninguna regla técnica del protocolo ni del código — abusan de reglas de negocio específicas de cada aplicación, que solo un auditor humano, entendiendo el propósito real del sistema, puede identificar como anómalas.
