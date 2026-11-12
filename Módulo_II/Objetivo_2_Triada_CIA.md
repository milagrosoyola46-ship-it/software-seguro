# Objetivo 2: Análisis de la Triada CIA

La seguridad de la información se apoya en tres pilares fundamentales, conocidos como la Triada CIA (Confidentiality, Integrity, Availability).

## Confidencialidad

La confidencialidad tiene que ver con que la información solo pueda ser vista o accedida por las personas que están autorizadas a hacerlo, y nadie más. Es asegurarse de que los datos sensibles (contraseñas, datos personales, información financiera, etc.) no caigan en manos de alguien que no debería tener acceso a ellos.

**Ejemplo de vulnerabilidad que compromete la confidencialidad: IDOR (Insecure Direct Object Reference)**

Un IDOR ocurre cuando una aplicación expone directamente un identificador (como el ID de un usuario o de un pedido) en una URL o en una petición, y no valida correctamente si el usuario que está pidiendo ese recurso tiene realmente permiso para verlo. Por ejemplo, si entro a `miapp.com/pedido?id=1001` y veo mi pedido, pero cambiando el número a `id=1002` puedo ver el pedido de otra persona (con su dirección, su nombre, lo que compró), ahí se está rompiendo la confidencialidad: estoy accediendo a información que no me corresponde ver, simplemente porque el sistema no comprobó si tenía autorización para ese recurso en particular.

## Integridad

La integridad se refiere a que la información se mantenga exacta y completa, sin que sea modificada, alterada o corrompida de forma no autorizada, ya sea por error o de manera intencional. Significa que los datos sean confiables: que lo que yo guardé o envié sea lo mismo que se recibe o se lee después, sin cambios indebidos en el medio.

**Ejemplo de vulnerabilidad que compromete la integridad: Mass Assignment / modificación de datos vía API**

Este es el mismo tipo de vulnerabilidad que encontré en el laboratorio de "Gran Rifa 2019": un endpoint de la API aceptaba directamente un campo como `esta_pago`, permitiendo que se modificara su valor desde el cliente sin ninguna validación de por medio. Esto compromete la integridad de los datos, porque alguien sin autorización real pudo alterar un dato importante (el estado de pago de una rifa) de una forma que no reflejaba la realidad: el sistema terminó "creyendo" algo que no era cierto, simplemente porque el dato fue modificado sin control.

## Disponibilidad

La disponibilidad implica que la información y los sistemas estén accesibles y funcionando correctamente cuando los usuarios autorizados los necesitan. No sirve de nada tener datos confidenciales y con integridad si, al momento de necesitarlos, el sistema está caído o inaccesible.

**Ejemplo de vulnerabilidad que compromete la disponibilidad: Ataque de Denegación de Servicio (DoS / DDoS)**

Un ataque de Denegación de Servicio busca saturar un servidor o una aplicación con una cantidad enorme de peticiones (ya sea desde una sola máquina, en el caso de DoS, o desde muchas máquinas distribuidas, en el caso de un DDoS), con el objetivo de que el sistema no pueda responder a las peticiones legítimas de los usuarios reales. El resultado es que la página o el servicio queda caído o extremadamente lento, y los usuarios que sí deberían tener acceso no pueden usarlo. Este tipo de ataque no busca robar ni modificar información, sino directamente tirar abajo la disponibilidad del servicio.

## Conclusión

Estos tres pilares están interrelacionados, y una aplicación segura necesita cuidarlos a los tres: no sirve de nada que un sistema esté siempre disponible si cualquiera puede ver información confidencial de otro usuario, ni sirve tener los datos bien protegidos y confidenciales si cualquiera puede modificarlos sin control. Analizar una vulnerabilidad pensando en cuál de estos tres pilares compromete ayuda a entender mejor el impacto real que tiene esa falla de seguridad.
