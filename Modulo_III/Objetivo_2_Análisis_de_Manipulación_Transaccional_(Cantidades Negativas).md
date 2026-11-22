# Objetivo 2: Análisis de Manipulación Transaccional (Cantidades Negativas)

## El escenario

Un e-commerce permite agregar productos al carrito. Al interceptar la petición (por ejemplo, con Burp Repeater, igual que hicimos en el Objetivo 2 del primer módulo), se modifica el campo `cantidad` de `1` a `-5`. El servidor acepta el valor y lo procesa sin rechazarlo, y el total del carrito termina siendo **negativo** — en vez de cobrarle al usuario, el sistema le **suma saldo a su cuenta**.

## Por qué ocurre esta falla técnicamente

### 1. El servidor confía ciegamente en un dato enviado por el cliente
El campo `cantidad` llega en la petición HTTP (ya sea en el body de un POST, o en un parámetro de URL) como un valor que el **cliente controla por completo** — el navegador, o en este caso Burp, puede enviar literalmente cualquier valor en ese campo, sin que exista ninguna restricción real hasta que el servidor decida qué hacer con él. Si el servidor simplemente toma ese número y lo usa directamente en su lógica de cálculo, sin verificar que tenga sentido dentro del dominio del negocio (una cantidad de productos **nunca puede ser negativa**), está delegando una regla de negocio crítica al cliente, que es exactamente el mismo patrón de falla que vimos con el precio en el Objetivo 1 del módulo anterior (Parameter Tampering).

### 2. La matemática simplemente "funciona" con números negativos
Esta es la parte más interesante del caso: no hay ningún error de sistema, ni ninguna excepción, ni ningún crash — el cálculo es perfectamente válido desde el punto de vista puramente matemático/aritmético:

```
total = precio_unitario * cantidad
total = $500 * (-5)
total = -$2500
```

El lenguaje de programación y el motor de base de datos no tienen ningún problema en operar con números negativos — la multiplicación es una operación matemática universal. El error **no está en la operación matemática en sí**, está en que el servidor nunca validó que `cantidad` perteneciera al dominio válido del negocio (números enteros **positivos**, mayores a cero) antes de usarla en el cálculo. Es, de nuevo, una **vulnerabilidad de Lógica de Negocio** (como vimos en el cuestionario del módulo anterior): la aplicación hizo exactamente lo que el código le decía que hiciera, pero el diseño de la regla de negocio no contempló (ni bloqueó) una entrada fuera del rango esperado.

### 3. El efecto cascada: de "carrito con total negativo" a "saldo a favor"
Si el sistema de checkout simplemente resta el `total` del carrito del medio de pago (o lo suma al saldo de la cuenta, dependiendo de cómo esté implementado el flujo), un total negativo invierte la operación: en vez de **cobrar** $2500, el sistema **acredita** $2500 a la cuenta del atacante. Esto puede materializarse como saldo a favor, un reembolso real a una tarjeta, o puntos/créditos de fidelización, dependiendo de qué sistema de pago esté detrás.

## Cómo debería validar esto el desarrollador, estrictamente, del lado del servidor

La validación del lado del cliente (JavaScript que evita escribir un número negativo en el input) **no sirve de nada** como defensa real — es trivial de saltar interceptando la petición con una herramienta como Burp, exactamente como hizo el atacante en este escenario. Toda la validación que importa tiene que ocurrir **en el servidor**, de forma redundante en varias capas:

### 1. Validación de rango en la capa de entrada (input validation)
Antes de procesar cualquier lógica, el servidor debe verificar explícitamente:
```python
if cantidad <= 0 or cantidad > LIMITE_MAXIMO_RAZONABLE:
    raise ValidationError("Cantidad inválida")
```
Esto debe aplicarse en **cada endpoint** que reciba este parámetro, no solo en el formulario "visible" del carrito — incluyendo APIs internas, endpoints de edición de cantidad, y cualquier otro punto de entrada que modifique el carrito.

### 2. Restricciones a nivel de tipo de dato y base de datos
Usar tipos de datos sin signo (`unsigned integer`) donde el lenguaje/base de datos lo permita, y agregar una **restricción CHECK** a nivel de base de datos:
```sql
ALTER TABLE carrito_items ADD CONSTRAINT chk_cantidad_positiva CHECK (cantidad > 0);
```
Esto actúa como una **última línea de defensa**: incluso si por algún motivo la validación de la aplicación falla o se omite en algún punto del código, la base de datos rechaza directamente el registro.

### 3. Nunca confiar en el total enviado por el cliente (si aplica)
Si además el cliente llegara a enviar también el `total` calculado (no solo la cantidad), el servidor debe **ignorarlo por completo** y recalcularlo siempre él mismo, a partir del precio real (obtenido de la base de datos, no del cliente) multiplicado por una cantidad ya validada como positiva — el mismo principio que vimos para el Parameter Tampering de precios.

### 4. Validación semántica adicional, no solo de signo
No alcanza con solo bloquear negativos: también conviene validar que la cantidad sea un número entero (no decimal, salvo que el producto se venda a granel), que no exceda el stock disponible, y que no supere límites razonables por pedido (para prevenir otros abusos, como acaparamiento o ataques de denegación de servicio por carritos gigantes).

## Conclusión

Esta falla es un ejemplo clásico de **confiar en datos del cliente para una regla de negocio crítica**: el sistema no tiene ningún error técnico (la aritmética con negativos es válida), pero nunca verificó que el dominio del negocio —una cantidad de productos— solo tiene sentido como un entero positivo. La única defensa real es validar estrictamente del lado del servidor, en cada capa posible (aplicación, base de datos), sin depender nunca de restricciones puramente visuales o de JavaScript en el cliente, que un atacante puede evadir con solo interceptar la petición.
