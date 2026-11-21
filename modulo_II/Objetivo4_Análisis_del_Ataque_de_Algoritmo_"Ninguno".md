# Objetivo 4: Análisis del Ataque de Algoritmo "None"

## ¿Qué es un JWT y cómo se verifica?

Un **JWT (JSON Web Token)** es un token compuesto por tres partes, separadas por puntos, cada una codificada en Base64URL:

```
header.payload.signature
```

- **Header:** indica el tipo de token y, crucialmente, el **algoritmo de firma** usado, en el campo `"alg"`. Ejemplo:
  ```json
  { "alg": "HS256", "typ": "JWT" }
  ```
- **Payload:** contiene los "claims" (datos), por ejemplo `{"user": "milagros", "role": "user"}`.
- **Signature:** el resultado de firmar `base64url(header) + "." + base64url(payload)` con una clave secreta (en HMAC) o una clave privada (en RSA/ECDSA).

Cuando el servidor recibe un JWT, normalmente hace dos cosas: (1) decodifica el header y el payload, y (2) **verifica la firma** usando el algoritmo y la clave que el servidor espera, para confirmar que el token no fue alterado y que fue emitido por el propio servidor.

## El algoritmo "none"

La especificación de JWT (RFC 7519 / JWS RFC 7515) define un algoritmo especial llamado **`"none"`**, pensado para casos legítimos donde el token no necesita integridad firmada (por ejemplo, un JWT ya protegido por otro mecanismo de transporte). Cuando `alg` es `"none"`, el JWT **no lleva firma** — el tercer segmento queda vacío.

El problema aparece cuando una **librería o implementación del backend** decide qué algoritmo usar para verificar la firma **leyendo el campo `alg` del propio token recibido**, en lugar de tener fijo de antemano qué algoritmo espera. Es decir, el servidor le pregunta al token "¿con qué algoritmo te verifico?" — y el token es generado por quien sea, incluido un atacante.

## Por qué esto permite falsificar un token sin conocer la clave secreta

El ataque funciona así:

1. El atacante intercepta o posee un JWT legítimo, por ejemplo:
   ```
   eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyIjoibWlsYWdyb3MiLCJyb2xlIjoidXNlciJ9.<firma_HS256>
   ```
2. Decodifica el header y el payload (son solo Base64, no están cifrados, cualquiera los puede leer).
3. Modifica el payload a su gusto, por ejemplo cambiando `"role":"user"` por `"role":"admin"`.
4. Cambia el header para que diga:
   ```json
   { "alg": "none", "typ": "JWT" }
   ```
5. Vuelve a codificar header y payload en Base64URL, y **deja el tercer segmento (la firma) vacío**:
   ```
   eyJhbGciOiJub25lIn0.eyJ1c2VyIjoibWlsYWdyb3MiLCJyb2xlIjoiYWRtaW4ifQ.
   ```
6. Envía este token al servidor.

Si el backend es vulnerable (lee el `alg` del token y, al ver `"none"`, directamente **salta el paso de verificación de firma**), el servidor va a aceptar el payload modificado como válido — **sin que el atacante haya necesitado conocer la clave secreta en ningún momento**, porque nunca se ejecutó ninguna operación criptográfica que dependiera de esa clave. El atacante no "rompió" la firma: logró que el servidor **no la pidiera**.

Esto le da al atacante **falsificación de tokens sin límite de tiempo y sin restricción**: puede generar un token con cualquier payload (cualquier usuario, cualquier rol, cualquier permiso) en el momento que quiera, de forma indefinida, porque el proceso de creación de un JWT "none" válido no requiere ningún secreto — solo requiere que el backend acepte ese algoritmo.

## Causa raíz del problema

La causa raíz **no es un defecto criptográfico** (no se rompe ningún HMAC ni RSA) sino un **error de diseño en la lógica de verificación**: confiar en un dato controlado por el atacante (el `alg` del header, que viaja dentro del propio token que se está validando) para decidir **cómo** se va a validar la seguridad de ese mismo token. Es un caso del patrón general "nunca dejes que la entrada no confiable decida la política de seguridad que se aplica sobre ella misma".

Esta misma clase de problema (confusión de algoritmo) tiene otras variantes conocidas, como el ataque **RS256 → HS256**: si un servidor usa RSA (clave pública/privada) para firmar, pero al verificar permite que el token diga `"alg":"HS256"`, un atacante que conozca la clave **pública** (que no es secreta, se distribuye abiertamente) puede usarla como clave HMAC para firmar un token falso, y el servidor lo valida como legítimo.

## Mitigación

- El servidor **nunca debe leer el algoritmo desde el token** para decidir cómo verificarlo. Debe tener **hardcodeado/configurado de antemano** qué algoritmo(s) espera (ej: "solo acepto HS256" o "solo acepto RS256") y rechazar cualquier token que declare un algoritmo distinto, incluyendo `"none"`.
- La mayoría de las librerías modernas de JWT ya corrigieron este problema por defecto, pero sigue siendo responsabilidad de quien integra la librería especificar explícitamente el/los algoritmo(s) permitido(s) al momento de verificar (allowlist), y no dejar ese parámetro librado al token recibido.

## 6. Conclusión

El ataque de algoritmo `"none"` es crítico porque convierte un mecanismo pensado para garantizar integridad (la firma del JWT) en papel mojado: si el backend confía en el `alg` que el propio atacante puede escribir, cualquiera puede emitir tokens arbitrarios — con cualquier usuario, rol o permiso, sin fecha de expiración forzada por falta de validación, y sin necesidad de romper ni conocer la clave secreta del servidor, porque esa clave nunca llega a usarse en la verificación.
