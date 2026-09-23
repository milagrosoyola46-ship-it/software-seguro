# Objetivo 4 - Instalacion Proxy
- Qué es un proxy?

Un proxy es un servidor o programa intermediario que se ubica entre el cliente (en este caso, el navegador) y el servidor de destino. En vez de que la petición viaje directamente del navegador al servidor, primero pasa por el proxy, que puede leerla, registrarla o incluso modificarla, y luego la reenvía. Lo mismo ocurre con la respuesta: primero llega al proxy y después al cliente.

- Diferencias principales entre un proxy y una VPN:

Alcance: el proxy actúa a nivel de aplicación, es decir, solo maneja el tráfico de lo que esté configurado para pasar por él (por ejemplo, el HTTP/HTTPS del navegador). La VPN actúa a nivel de todo el sistema operativo, encapsulando absolutamente todo el tráfico de red del dispositivo.

Objetivo principal: un proxy sirve para inspeccionar, filtrar o modificar peticiones puntuales. Una VPN, crea un túnel cifrado para proteger y anonimizar toda la conexión hacia una red remota.

Visibilidad del contenido: el proxy puede leer el contenido en texto plano. En una VPN, el contenido viaja cifrado dentro del túnel, y ni siquiera el proveedor de la red local puede leerlo.

Uso típico en seguridad: el proxy se usa sobre todo en pentesting, debugging de aplicaciones web, o control de acceso corporativo. La VPN se usa más para proteger la privacidad del usuario, acceder de forma remota a redes internas, o evadir restricciones geográficas.

  
