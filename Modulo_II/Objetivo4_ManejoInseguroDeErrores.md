# Objetivo 4: Manejo Inseguro de Errores (Stack Traces)

## Mensaje de error genérico vs. Stack Trace detallado

Cuando una aplicación falla, hay dos formas típicas de responderle al usuario:

### Mensaje de error genérico
```json
{
  "error": "Ocurrió un error interno. Por favor, intentá más tarde.",
  "code": "INTERNAL_SERVER_ERROR",
  "request_id": "7f3a9b2c-1e4d-4a8f-9c3e-5b6d7a8e9f01"
}
```
No revela nada sobre el funcionamiento interno del sistema. Como mucho, incluye un `request_id` para que soporte técnico pueda buscar el error real en los logs internos (que el usuario no ve).

### Stack Trace expuesto (incorrecto en producción)
```
Traceback (most recent call last):
  File "/app/routes/users.py", line 42, in get_user_profile
    user = db.session.query(User).filter(User.id == user_id).one()
  File "/usr/local/lib/python3.11/site-packages/sqlalchemy/orm/query.py", line 2854, in one
    raise orm_exc.NoResultFound("No row was found when one was required")
sqlalchemy.orm.exc.NoResultFound: No row was found when one was required

Flask app running on: /app/main.py
Database: postgresql://app_user:********@10.0.2.15:5432/prod_db
Debug mode: True
```
Este tipo de respuesta sale directamente de la aplicación (o del framework) cuando ocurre una excepción no controlada y el entorno de errores detallados (ej. `DEBUG=True`) queda activo en producción.

## Qué información le da esto a un atacante

Un stack trace detallado, es una fuente enorme de **reconocimiento pasivo**, y acelera drásticamente la fase de explotación porque le entrega al atacante, gratis, datos que de otra forma tendría que adivinar o fuerza bruta:

- **Tecnología y versión exacta:** el framework (`Flask`, `Django`, `Spring`, etc.), el lenguaje y versión (`Python 3.11`), y las librerías usadas (`SQLAlchemy`), lo que le permite buscar directamente **CVEs conocidos** para esa versión específica, en vez de tener que adivinar la tecnología.
- **Estructura interna del código:** rutas de archivos (`/app/routes/users.py`), nombres de funciones (`get_user_profile`) y la lógica del backend — esto le muestra al atacante **cómo está armada la aplicación por dentro**, qué endpoints existen, qué módulos la componen, y en muchos casos cómo construir un ataque dirigido (por ejemplo, saber que hay una consulta tipo `filter(User.id == user_id)` sugiere que se podría probar alterando el `user_id` — una pista directa hacia un IDOR).
- **Configuración del entorno de infraestructura:** rutas absolutas del servidor (`/app/main.py`), direcciones IP internas (`10.0.2.15`), nombres de bases de datos (`prod_db`), tipo de motor de base de datos (`postgresql`) y en algunos casos (configuraciones peores) hasta **credenciales parciales o completas** en la cadena de conexión.
- **Confirmación de que el modo debug está activo:** `Debug mode: True` es en sí mismo una vulnerabilidad grave en muchos frameworks (en Flask/Django, el modo debug puede incluso permitir ejecución remota de código a través de la consola interactiva de depuración si el atacante logra disparar un error).
- **Validación de hipótesis de ataque:** si el atacante está probando SQL Injection, un stack trace que muestre el motor de base de datos y la sintaxis exacta de la consulta fallida le confirma inmediatamente que su payload llegó a la base de datos y le da pistas de cómo ajustar la inyección (esto es la base de la técnica conocida como **Error-Based SQL Injection**).

## Por qué esto acelera la fase de explotación

En la metodología de un pentest (reconocimiento → escaneo → explotación → post-explotación), normalmente el atacante tiene que invertir tiempo y esfuerzo en la fase de reconocimiento para descubrir: qué tecnología corre el servidor, qué versión, cómo está estructurado el código, y qué errores producen qué comportamientos.

Un stack trace expuesto **le regala ese reconocimiento gratis y con alta precisión** — no son suposiciones basadas en fingerprinting externo (headers, banners), sino información exacta y confirmada directamente por el propio servidor. Esto reduce drásticamente el tiempo entre "encontré la aplicación" y "sé exactamente qué atacar y cómo", porque:

1. Elimina la necesidad de adivinar la pila tecnológica (fingerprinting).
2. Permite buscar exploits públicos ya documentados para la versión exacta detectada.
3. Expone detalles de implementación (nombres de tablas, columnas, rutas de archivos, lógica de negocio) que de otra forma requerirían fuzzing extenso para descubrir.
4. Confirma en tiempo real si un payload de ataque (SQLi, path traversal, etc.) tuvo efecto, funcionando como un "oráculo" que le dice al atacante si va por buen camino.

## Mitigación

- En producción, **nunca** debe estar activo el modo debug/desarrollo del framework (`DEBUG=False` en Django/Flask, `NODE_ENV=production` en Node, etc.).
- Implementar un **manejador de errores global (error handler)** que capture cualquier excepción no controlada y devuelva siempre un mensaje genérico al cliente.
- Registrar el stack trace completo **solo en los logs del servidor** (nunca en la respuesta HTTP), idealmente con un sistema de monitoreo (Sentry, Datadog, CloudWatch, etc.) y un `request_id` correlacionable para debugging interno.
- Separar claramente los entornos: lo que se ve en desarrollo/staging nunca debe ser lo que se expone en producción.

## Conclusión

La diferencia entre un mensaje de error genérico y un stack trace expuesto no es solo estética: un stack trace convierte cada error de la aplicación en una fuga de información que le ahorra al atacante gran parte del trabajo de reconocimiento, revelando tecnología, versión, estructura de código, configuración de infraestructura y, en el peor de los casos, confirmación en vivo de si sus intentos de explotación (SQLi, path traversal, etc.) están funcionando.
