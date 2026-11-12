# Objetivo 3: Análisis de Automatización y Fuerza Bruta

## Diferencia entre Fuerza Bruta y Ataque de Diccionario

### Ataque de Fuerza Bruta

Un ataque de fuerza bruta consiste en probar sistemáticamente **todas las combinaciones posibles** de caracteres hasta dar con la contraseña correcta. No se basa en ninguna suposición sobre cuál podría ser la contraseña: simplemente va generando combinaciones de forma exhaustiva, empezando por las más cortas y simples, y aumentando la complejidad a medida que avanza (por ejemplo, primero prueba "a", "b", "c"..., después "aa", "ab", "ac"..., y así sucesivamente).

La principal característica de este tipo de ataque es que, en teoría, siempre termina encontrando la contraseña correcta, porque recorre todas las combinaciones posibles. El problema es que, cuanto más larga y compleja sea la contraseña (con mayúsculas, números, símbolos), más tiempo lleva encontrarla, al punto de que para contraseñas suficientemente largas puede volverse inviable, incluso con mucho poder de cómputo, porque la cantidad de combinaciones crece de forma exponencial.

### Ataque de Diccionario

Un ataque de diccionario, en cambio, no prueba combinaciones al azar, sino que utiliza una **lista predefinida de palabras y contraseñas comunes** (el "diccionario"), que puede incluir palabras del idioma, nombres propios, fechas, o contraseñas que se filtraron en brechas de seguridad anteriores y que mucha gente sigue reutilizando (como "123456", "contraseña", "qwerty", nombres de mascotas, etc.).

Este tipo de ataque es mucho más rápido y eficiente que la fuerza bruta pura, porque en vez de probar todas las combinaciones posibles, se enfoca directamente en las contraseñas que la gente realmente usa con más frecuencia. La desventaja es que, si la contraseña real no está en esa lista (por ejemplo, si es una combinación aleatoria generada por un gestor de contraseñas), el ataque de diccionario no la va a encontrar nunca, por más tiempo que corra.

### La diferencia fundamental

En resumen, la fuerza bruta prueba **todo**, sin ningún criterio, mientras que el diccionario prueba **lo más probable primero**, basándose en patrones reales de cómo la gente arma sus contraseñas. En la práctica, muchas herramientas combinan ambos enfoques: primero prueban un diccionario de contraseñas comunes, y si eso falla, recién ahí recurren a la fuerza bruta pura como último recurso.

## Mecanismo de defensa: Rate Limiting (limitación de intentos)

Un mecanismo efectivo que un desarrollador puede implementar para mitigar tanto ataques de fuerza bruta como de diccionario es el **rate limiting**, que consiste en limitar la cantidad de intentos de login que se pueden hacer en un período de tiempo determinado, ya sea por cuenta de usuario, por dirección IP, o por ambos.

Por ejemplo, se puede configurar el sistema para que, después de 5 intentos fallidos de login en un minuto, bloquee temporalmente esa cuenta o esa IP durante unos minutos antes de permitir intentarlo de nuevo (esto se suele combinar con un "backoff" progresivo, donde cada bloqueo sucesivo dura más tiempo que el anterior). Esto hace que un ataque automatizado, que depende de poder probar miles o millones de combinaciones en poco tiempo, se vuelva completamente inviable en la práctica, porque el tiempo que tardaría en completar el ataque se vuelve absurdamente largo.

Otras medidas complementarias que suelen implementarse junto con el rate limiting son los CAPTCHA (para distinguir a un humano de un script automatizado) y la autenticación de dos factores (2FA), que hace que, aunque un atacante logre adivinar la contraseña correcta, igual no pueda acceder sin el segundo factor de verificación.

## Conclusión

Tanto la fuerza bruta como los ataques de diccionario buscan lo mismo (adivinar una credencial válida), pero con estrategias distintas: una exhaustiva y sin criterio, la otra basada en patrones reales de uso. Entender esta diferencia ayuda a elegir las defensas correctas: mientras que el rate limiting es efectivo contra ambos tipos de ataque al limitar la velocidad de prueba, exigir contraseñas largas y complejas es más efectivo específicamente contra la fuerza bruta, y educar a los usuarios para que no usen contraseñas comunes ayuda a mitigar los ataques de diccionario.
