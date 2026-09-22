# Objetivo 1 - Mapeo Lógico
## Aplicación web seleccionada
**Gran Rifa 2019** — laboratorio de la plataforma Software Seguro (`softwareseguro.com.ar`). Es un sitio web tradicional que simula un sistema de venta de rifas, con pantalla de login (usuario/clave) y funcionalidades de compra. Representa el caso típico de una aplicación con frontend, backend y base de datos separados, ideal para mapear el recorrido de un dato desde que el usuario inicia sesión hasta que esa información se persiste.

## Diagrama de flujo del dato

```
[Usuario] --(mismo dispositivo)--> [Frontend]
                                        |
                                        |  HTTPS/TLS (cifrado)
                                        v
                                   [Backend]
                                        |
                                        |  Red interna / VPC
                                        v
                                [Base de datos]
```

1. **Usuario → Frontend**: el usuario interactúa con la interfaz web (HTML/JS/Angular) directamente en su navegador. Este tramo ocurre en el mismo dispositivo, no hay red de por medio.
2. **Frontend → Backend**: el navegador envía la petición (por ejemplo, el login con usuario `guido` y clave `RIFA_2019`, o la compra de un número de rifa) a través de Internet hacia el servidor. **Este es el tramo exacto donde actúa HTTPS**, cifrando los datos en tránsito mediante TLS para evitar que un atacante en la red (man-in-the-middle) pueda leer o modificar la información.
3. **Backend → Base de datos**: el servidor procesa la petición (valida, autentica, aplica lógica de negocio) y consulta o escribe en la base de datos. Este tramo suele ocurrir dentro de una red interna o privada (LAN/VPC) y normalmente **no usa HTTPS**, sino el protocolo propio de la base de datos (a veces con TLS interno, pero es una capa distinta).

## ¿Dónde protege exactamente HTTPS?

HTTPS protege **la comunicación entre el cliente (navegador) y el servidor (backend)**, es decir, el tramo que atraviesa Internet. No protege:
- Los datos una vez que ya están dentro del servidor (en memoria o en logs).
- La comunicación interna entre el backend y la base de datos, salvo que se configure TLS de forma explícita en esa conexión.
- El dispositivo del usuario en sí (si su equipo está comprometido, HTTPS no ayuda).

## Relevancia para el pentesting

Entender este mapa es clave porque cada tramo tiene una superficie de ataque distinta:
- **Frontend**: XSS, manipulación del DOM, exposición de código cliente.
- **Tránsito Frontend-Backend**: si no hay HTTPS (o está mal configurado), es vulnerable a sniffing, downgrade attacks o MITM.
- **Backend**: inyección de comandos, lógica de negocio insegura, autenticación/autorización.
- **Backend-Base de datos**: SQL injection, credenciales hardcodeadas, falta de cifrado en tránsito interno.
nd-Backend**: si no hay HTTPS (o está mal configurado), es vulnerable a sniffing, downgrade attacks o MITM.
- **Backend**: inyección de comandos, lógica de negocio insegura, autenticación/autorización.
- **Backend-Base de datos**: SQL injection, credenciales hardcodeadas, falta de cifrado en tránsito interno.
