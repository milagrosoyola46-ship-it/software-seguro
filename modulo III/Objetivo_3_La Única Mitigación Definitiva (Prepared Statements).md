# Objetivo 3: La Única Mitigación Definitiva (Prepared Statements)

## Por qué las "listas negras" (blacklisting) fracasan

El enfoque intuitivo de muchos desarrolladores es tratar la inyección SQL como un problema de "texto prohibido": bloquear palabras como `SELECT`, `UNION`, `DROP`, o escapar comillas simples (`'` → `\'`). Esto falla estructuralmente por varias razones:

- **Es una lista de lo conocido, no de lo posible.** Siempre existe una forma de reescribir un payload que no esté en la lista: usar mayúsculas/minúsculas mezcladas (`SeLeCt`), comentarios inline para partir la palabra (`UN/**/ION`), codificaciones alternativas (hex, Unicode), o sinónimos funcionales del motor de base de datos que logran el mismo efecto sin usar la palabra bloqueada.
- **Escapar comillas no cubre todos los contextos de inyección.** Si el dato se inserta en un contexto numérico (`WHERE id = 5` sin comillas), escapar comillas no sirve de nada — el atacante ni siquiera necesita usar una comilla para inyectar (`id=5 OR 1=1`).
- **Es un parche a nivel de "texto", mientras el problema es a nivel de "estructura".** El bloqueo de palabras intenta adivinar la intención del atacante mirando el string crudo, en vez de resolver la causa raíz: que el dato del usuario termina **mezclado dentro del mismo canal** que el código SQL que define la estructura de la consulta.

## La causa raíz real: mezclar código y datos en el mismo string

Una consulta SQL tradicional, construida por concatenación de strings, se ve así:

```python
query = "SELECT * FROM usuarios WHERE nombre = '" + nombre_usuario + "'"
```

El problema de fondo es que, para el motor de base de datos, **todo lo que llega en ese string es una sola cosa**: no hay forma de distinguir, una vez concatenado, qué parte era "la instrucción que escribió el desarrollador" y qué parte era "el dato que escribió el usuario". Si el usuario envía `' OR '1'='1`, al concatenarse, el motor de base de datos recibe:

```sql
SELECT * FROM usuarios WHERE nombre = '' OR '1'='1'
```

y la ejecuta tal cual, sin ninguna forma de saber que esa porción extra de SQL no era parte de la intención original del desarrollador — **para el motor, es simplemente SQL válido**, indistinguible de cualquier otra parte de la consulta.

## Cómo funciona una Consulta Parametrizada (Prepared Statement) a nivel de arquitectura

Una consulta parametrizada resuelve esto separando **completamente** el código SQL de los datos, en dos pasos distintos y en dos canales distintos de comunicación con la base de datos:

### Paso 1 — Preparación (Prepare)
La aplicación envía primero al motor de base de datos **solo la estructura de la consulta**, con marcadores de posición (placeholders) en lugar de los valores reales:

```sql
SELECT * FROM usuarios WHERE nombre = ?
```
(o `%s`, `:nombre`, según el driver/lenguaje)

En este paso, el motor de base de datos **ya compila y analiza la consulta completa**: define el plan de ejecución, entiende exactamente qué tabla, qué columnas, y qué estructura lógica tiene la consulta — **sin haber recibido todavía ningún dato del usuario**. En este punto, la estructura de la consulta queda "congelada" — no hay forma de modificarla después.

### Paso 2 — Ejecución (Execute) con los parámetros enviados por separado
Recién en un segundo paso, completamente separado, la aplicación envía los valores reales:

```python
cursor.execute("SELECT * FROM usuarios WHERE nombre = ?", (nombre_usuario,))
```

Estos valores **no se concatenan como texto a la consulta** — se envían al motor de base de datos por un canal de datos aparte (en el protocolo de comunicación del driver), y el motor los trata **exclusivamente como el valor literal de ese parámetro específico**, sin importar qué caracteres contengan.

## Por qué esto neutraliza la inyección por completo, sin importar qué envíe el usuario

La clave está en que, para cuando el motor de base de datos recibe el valor del parámetro, **ya no está "leyendo SQL"** en esa posición — ya terminó de interpretar y compilar la estructura de la consulta en el Paso 1. El valor que llega en el Paso 2 se trata como un **dato atómico**, sea cual sea su contenido: si el atacante envía `' OR '1'='1`, el motor no lo interpreta como "cerrar comilla, agregar condición OR, abrir comilla" — lo interpreta literalmente como la cadena de texto de 15 caracteres `' OR '1'='1`, y busca en la tabla un registro cuyo campo `nombre` sea exactamente igual a ese texto completo (comillas incluidas), que obviamente no existe, devolviendo cero resultados — sin ejecutar ninguna lógica adicional.

No hay ningún carácter especial, ninguna palabra clave SQL, ninguna codificación alternativa que el atacante pueda enviar que logre "escapar" de ese tratamiento, porque **la estructura de la consulta ya no puede modificarse en esta etapa** — el motor ya decidió, en el Paso 1, exactamente qué va a hacer con ese parámetro (compararlo contra la columna `nombre`), y ningún valor en el Paso 2 puede alterar esa decisión ya tomada.

## Por qué esto es "definitivo" y no solo "otra mitigación más"

A diferencia de un WAF, un filtro de blacklist, o un escape manual de caracteres —que son todas defensas que intentan **adivinar e interceptar patrones maliciosos conocidos**, y que por lo tanto siempre pueden tener un caso límite no contemplado—, las consultas parametrizadas **eliminan la posibilidad estructural de la inyección**: no es que detecten y bloqueen el ataque, es que el canal por el cual un ataque de este tipo podría viajar (la mezcla de código y datos en un mismo string) directamente **deja de existir**. No hay nada que "evadir", porque no hay ningún punto en el proceso donde el dato del usuario pueda influir en la estructura de la consulta.

## Conclusión

Las listas negras y el escape de caracteres tratan la inyección SQL como un problema de "texto peligroso a filtrar", una carrera interminable contra formas cada vez más creativas de ofuscar un payload. Las consultas parametrizadas resuelven el problema en su raíz arquitectónica: separan por completo la estructura de la consulta (definida y compilada antes de ver ningún dato del usuario) de los valores (tratados siempre como datos literales, nunca como código), haciendo que el contenido del input del usuario sea, por diseño, irrelevante para la lógica SQL que se ejecuta — sin importar qué caracteres, palabras clave o codificaciones intente usar un atacante.
