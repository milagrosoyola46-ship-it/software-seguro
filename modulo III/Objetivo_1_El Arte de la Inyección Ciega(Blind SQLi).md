# Objetivo 1: El Arte de la Inyección Ciega (Blind SQLi)

## Por qué existe la "inyección ciega"?

En una SQL Injection "normal" (Error-Based o Union-Based), la aplicación devuelve directamente en la respuesta algo útil para el atacante: un mensaje de error con la sintaxis SQL rota, o datos extra en la página gracias a un `UNION SELECT`. Pero muchas aplicaciones bien configuradas no muestran ni errores de base de datos ni resultados extra. Solo devuelven dos estados posibles para el usuario: una pantalla "normal" o, en el peor de los casos, nada visible distinto.

Cuando la aplicación es vulnerable a SQLi pero **no filtra directamente datos en la respuesta**, el atacante tiene que explotarla "a ciegas": inyectando condiciones lógicas y observando **indirectamente** cómo responde el servidor (cambia el contenido, o cambia el tiempo que tarda) para deducir, pregunta por pregunta, los datos reales de la base. Esto se llama **Blind SQL Injection**.

Hay dos variantes principales, según qué señal indirecta usa el atacante para "leer" la respuesta de la base de datos.

## Boolean-Based Blind SQLi

**Señal que usa el atacante:** un cambio **visible en el contenido** de la página (verdadero/falso), no un error ni datos extra.

**Cómo funciona:** el atacante inyecta una condición lógica que es verdadera o falsa, y observa si la página responde de forma distinta en cada caso. Por ejemplo, para un endpoint `GET /producto?id=5`:

```
id=5 AND 1=1   -->  la página carga normal (el producto existe y la condición es TRUE)
id=5 AND 1=2   -->  la página muestra "Producto no encontrado" (la condición es FALSE)
```

Si el atacante ve que la respuesta **cambia** entre esos dos casos (aunque nunca vea un error de SQL ni datos filtrados), confirma que la condición `AND` se está evaluando dentro de una consulta SQL real, es decir, confirma la inyección.

A partir de ahí, se puede extraer cualquier dato convirtiéndolo en una serie de preguntas de verdadero/falso. Por ejemplo, para saber el nombre de la base de datos:

```
id=5 AND SUBSTRING(DATABASE(),1,1)='a'   --> FALSE (página de "no encontrado")
id=5 AND SUBSTRING(DATABASE(),1,1)='b'   --> FALSE
...
id=5 AND SUBSTRING(DATABASE(),1,1)='t'   --> TRUE (página normal) ✔
```

Esta condición extrae el **primer carácter** del nombre de la base de datos (`SUBSTRING(DATABASE(),1,1)`) y lo compara letra por letra contra el abecedario (y dígitos/símbolos) hasta encontrar la que hace que la condición sea verdadera. Una vez encontrada la primera letra, se repite el proceso para la posición 2, la 3, y así sucesivamente, reconstruyendo el nombre completo carácter por carácter, únicamente observando si la página responde "normal" o "no encontrado" en cada intento.

## Time-Based Blind SQLi

**Señal que usa el atacante:** el **tiempo** que tarda el servidor en responder, no ningún cambio visible en el contenido.

**Cuándo se usa:** cuando ni siquiera hay una diferencia visible entre verdadero y falso (la página responde exactamente igual en ambos casos, por ejemplo porque el resultado de la consulta no afecta nada que el usuario vea, solo se ejecuta "en segundo plano"). En ese caso, Boolean-Based no sirve porque no hay ninguna señal observable en el contenido.

**Cómo funciona:** el atacante inyecta una condición que, si es verdadera, fuerza a la base de datos a **esperar X segundos artificialmente** antes de responder (usando funciones como `SLEEP()` en MySQL o `WAITFOR DELAY` en SQL Server):

```
id=5 AND IF(1=1, SLEEP(5), 0)   --> el servidor tarda 5 segundos de más en responder (condición TRUE)
id=5 AND IF(1=2, SLEEP(5), 0)   --> el servidor responde inmediatamente (condición FALSE)
```

El atacante simplemente **mide el tiempo de respuesta** con un cronómetro (o, en la práctica, con una herramienta automatizada): si tarda ~5 segundos extra, la condición era verdadera; si responde al instante, era falsa. No hace falta ver ningún cambio en el contenido de la página — el servidor podría devolver exactamente el mismo HTML en ambos casos, y aun así se puede distinguir la respuesta por el tiempo.

De la misma forma que en Boolean-Based, se puede extraer el nombre de la base de datos letra por letra:

```
id=5 AND IF(SUBSTRING(DATABASE(),1,1)='a', SLEEP(5), 0)   --> responde rápido (FALSE)
...
id=5 AND IF(SUBSTRING(DATABASE(),1,1)='t', SLEEP(5), 0)   --> tarda 5 segundos (TRUE) ✔
```

## Cómo se extrae el nombre completo, letra por letra, automatizado

En ambos casos, el proceso manual sería extremadamente lento (probar cada letra del abecedario, para cada posición del string, para cada carácter que se quiere extraer). En la práctica, un atacante automatiza esto con herramientas como **sqlmap**, que:

1. Prueba automáticamente miles de combinaciones de condiciones (`SUBSTRING`, comparaciones ASCII, etc.).
2. Usa **búsqueda binaria** en vez de probar letra por letra en orden (ej: en vez de probar `a, b, c, d...`, pregunta primero "¿el código ASCII del carácter es mayor a 97?" para descartar la mitad de las posibilidades en cada intento), reduciendo drásticamente la cantidad de peticiones necesarias.
3. Repite el proceso posición por posición (`SUBSTRING(DATABASE(),1,1)`, luego `SUBSTRING(DATABASE(),2,1)`, etc.) hasta reconstruir el string completo, ya sea observando el contenido (Boolean-Based) o el tiempo de respuesta (Time-Based).

## Diferencia técnica clave entre ambas

| | Boolean-Based | Time-Based |
|---|---|---|
| **Señal observada** | Cambio en el contenido/comportamiento visible de la respuesta | Cambio en el tiempo que tarda la respuesta |
| **Requiere que la app muestre algo distinto entre true/false** | Sí | No |
| **Funciona aunque la respuesta sea idéntica en todos los casos** | No | Sí |
| **Velocidad de extracción** | Más rápida (no depende de esperar segundos por cada intento) | Más lenta (cada intento falso-positivo puede tardar varios segundos) |
| **Detectabilidad en logs** | Igual de "silenciosa" en cuanto a errores, pero genera patrones de tráfico similares | Genera picos de latencia que pueden ser detectados por un WAF/monitoreo |

En resumen: **Boolean-Based** se usa cuando hay alguna diferencia visible en el contenido de la respuesta, y es más rápida; **Time-Based** es el último recurso cuando la aplicación responde exactamente igual en todos los casos, y el único canal que queda disponible para filtrar información, bit por bit, es el tiempo.

## Conclusión

Blind SQL Injection demuestra que una aplicación puede ser completamente vulnerable a inyección SQL **sin mostrar nunca un solo error ni un solo dato filtrado directamente en pantalla**. Mientras exista algún canal observable por el atacante —ya sea un cambio de contenido (Boolean-Based) o una variación de tiempo (Time-Based)— es posible reconstruir, carácter por carácter, cualquier dato de la base de datos: nombres de tablas, columnas, usuarios, contraseñas, etc. Por eso la defensa real no puede depender de "no mostrar errores": la única mitigación efectiva es usar **consultas parametrizadas (prepared statements)**, que eliminan la inyección en la capa de la consulta misma, sin importar qué señal pueda o no observar el atacante.
