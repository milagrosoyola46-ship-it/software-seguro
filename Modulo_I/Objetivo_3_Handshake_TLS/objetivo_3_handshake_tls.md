# Objetivo 3 - Handshake TLS

## Qué es el TLS Handshake

Básicamente es un intercambio de mensajes entre el navegador (cliente) y el servidor que ocurre antes de que se mande cualquier dato real de la página. La idea es que ambos se pongan de acuerdo en tres cosas: que el servidor sea realmente quien dice ser, qué forma de cifrado van a usar, y una clave secreta que solo ellos dos van a conocer para cifrar todo lo que se manden después.

## Cómo funciona, paso a paso

Primero el navegador manda un mensaje llamado "Client Hello", donde le dice al servidor qué versiones de TLS soporta y qué algoritmos de cifrado puede usar, además de mandar un número random que va a servir más adelante.

El servidor responde con un "Server Hello": elige la versión y el algoritmo de cifrado de la lista que le mandó el cliente, genera su propio número random, y lo más importante, le manda su certificado digital.

Ahí el navegador se pone a revisar ese certificado. Chequea que esté firmado por alguna entidad certificadora en la que confía, que no esté vencido, y que el dominio del certificado coincida con el sitio al que me estoy conectando. Si todo eso cierra, ahí el navegador sabe que el servidor es quien dice ser, y no alguien haciéndose pasar por él en el medio (lo que sería un ataque man-in-the-middle).

Una vez validado el certificado, ahí es donde entra el cifrado asimétrico: el navegador genera un valor secreto, lo cifra usando la clave pública que vino en el certificado del servidor, y se lo manda. Como solo el servidor tiene la clave privada que le corresponde a esa pública, es el único que puede descifrarlo. Con ese secreto más los números random que se mandaron antes, los dos lados calculan por separado la misma clave de sesión.

A partir de ahí dejan de usar cifrado asimétrico y pasan a cifrado simétrico con esa clave que ya comparten, y con eso se cifra toda la comunicación real que sigue. Para cerrar, ambos mandan un mensaje de confirmación cifrado avisando que ya está todo listo, y recién en ese momento el navegador manda la primera petición HTTP de verdad.

## Por qué se usan dos tipos de cifrado y no uno solo

El cifrado asimétrico (el de clave pública y privada) tiene la ventaja de que permite que dos partes que nunca se comunicaron antes puedan ponerse de acuerdo en un secreto de forma segura, sin tener que haber compartido una clave por otro lado. El problema es que es bastante pesado computacionalmente, porque usa operaciones matemáticas complejas, entonces si se usara para cifrar todo el tráfico de una sesión completa de navegación sería mucho más lento.

El cifrado simétrico en cambio es mucho más rápido y liviano, pero tiene el problema de que ambas partes necesitan tener la misma clave de antemano, y no hay forma segura de ponerse de acuerdo en esa clave si no se conocían antes.

Por eso TLS combina los dos: usa el cifrado asimétrico nada más que durante el handshake, para autenticar al servidor y acordar de forma segura esa clave de sesión, y después cambia a cifrado simétrico para el resto de la conexión, que es mucho más eficiente para mandar y recibir datos todo el tiempo que dura la sesión.

## El rol de los certificados

El certificado cumple dos funciones en todo esto. Por un lado sirve para autenticar al servidor, o sea, probar que es quien dice ser gracias a que está firmado por una entidad certificadora confiable. Por otro lado, es el medio por el cual el servidor le entrega al navegador su clave pública, que es la que se usa después para el intercambio de claves.

Si no existiera el certificado, cualquiera podría meterse en el medio de la conexión y hacerse pasar por el servidor real, entregando su propia clave pública y quedándose con la capacidad de leer o modificar todo lo que se manda. Por eso ese paso de verificación del certificado es tan importante dentro del proceso.

