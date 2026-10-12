## Puertos y Servicios Esenciales

## Puerto 20 y 21 — FTP (File Transfer Protocol)

El puerto 21 es el que usa FTP para el control de la conexión (login, comandos), mientras que el puerto 20 se usa para la transferencia de datos en modo activo. Es un protocolo viejo que permite subir y bajar archivos de un servidor, pero por defecto no cifra nada, así que las credenciales viajan en texto plano si no se usa una variante segura como FTPS o SFTP.

## Puerto 22 — SSH (Secure Shell)

Se usa para acceder de forma remota y segura a la terminal de un servidor. A diferencia de Telnet (su predecesor inseguro), SSH cifra toda la comunicación, incluidas las credenciales. También se usa mucho para transferir archivos de forma segura (SCP, SFTP) y para túneles.

## Puerto 23 — Telnet

Es el protocolo más viejo para acceso remoto a una terminal. El problema es que no cifra absolutamente nada: usuario, contraseña y todo el tráfico viajan en texto plano, por lo que hoy en día se considera inseguro y no se usa tanto, salvo en sistemas legacy o dispositivos de red muy antiguos.

## Puerto 25 — SMTP (Simple Mail Transfer Protocol)

Es el protocolo que se usa para enviar correos electrónicos entre servidores de mail. No se usa para leer el correo (eso lo hacen otros protocolos como IMAP o POP3), sino específicamente para el envío y el reenvío (relay) de mensajes entre servidores.

## Puerto 53 — DNS (Domain Name System)

Es el servicio que traduce nombres de dominio (como google.com) a direcciones IP. Funciona tanto sobre UDP como sobre TCP: UDP se usa para la mayoría de las consultas normales, más rápidas, y TCP se usa cuando la respuesta es muy grande o para transferencias de zona completas entre servidores DNS.

## Puerto 80 — HTTP

Es el puerto estándar para tráfico web sin cifrar. Cualquier dato que viaje por acá (incluidas credenciales si un sitio mal configurado las pide) puede ser interceptado y leído por cualquiera que esté en el medio de la comunicación.

## Puerto 110 — POP3 (Post Office Protocol v3)

Se usa para que un cliente de correo (como Outlook) descargue los mensajes desde el servidor. A diferencia de IMAP, POP3 normalmente descarga los correos al dispositivo y los borra del servidor, aunque esto depende de la configuración.

## Puerto 443 — HTTPS

Es el mismo tráfico web que el puerto 80, pero cifrado mediante TLS/SSL. Es el puerto que usan hoy en día prácticamente todos los sitios web serios, ya que protege la confidencialidad e integridad de los datos que viajan entre el navegador y el servidor.

## Puerto 3306 — MySQL

Es el puerto por defecto que usa el motor de base de datos MySQL para aceptar conexiones. En un pentest, encontrar este puerto abierto y accesible desde afuera suele ser una mala señal, porque una base de datos nunca debería estar expuesta directamente a internet sin restricciones.
