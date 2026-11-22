# Objetivo 2: Ingeniería Inversa a JSON Web Tokens (JWT)

## Interceptación del token

Inspeccionando las cookies del dominio `app.softwareseguro.com.ar`, identifique la cookie **`auth`**, el valor tiene la forma típica de un JWT: tres partes separados por puntos, el primero comenzando con `eyJ...`.

<img width="1568" height="613" alt="image" src="https://github.com/user-attachments/assets/a6142b4c-662e-462c-a40c-8f452a0db546" />

Estructura general de un JWT:

```
<header_base64>.<payload_base64>.<firma_base64>
```

Las dos primeras partes están codificadas en **Base64URL**, no cifradas — cualquiera que tenga el token puede decodificarlas sin necesitar ninguna clave. la tercera parte (la firma) requiere la clave secreta del servidor, pero esa firma únicamente sirve para *validar* el token, no para ocultar su contenido.

## Header decodificado

Decodificando en Base64URL el primer segmento se obtiene:

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

**Algoritmo de firma utilizado: `HS256`**.

<img width="1568" height="333" alt="image" src="https://github.com/user-attachments/assets/8da9f9d8-0f93-40fa-bac4-f9df3a6c06d1" />

## Payload decodificado

Decodificando el segundo segmento se obtiene un JSON con los datos del usuario (valores de ejemplo, reemplazando los datos reales por privacidad):

```json
{
  "id": 1088,
  "fullname": "Nombre Apellido",
  "email": "usuario@ejemplo.com",
  "role_id": 1,
  "plan_id": 2,
  "pagado": true,
  "fecha_pago": "2026-09-03T21:42:12.674Z",
  "plan_expires_at": null,
  "email_confirmed": true,
  "pending_payment": null,
  "confirmation_email_uuid": "4045d0ac-7fab-4138-a8c2-cf1fc60852d4",
  "created_at": "2026-09-20T23:56:27.277Z",
  "is_superadmin": false,
  "iat": 1795322562,
  "exp": 1795344162
}
```

<img width="790" height="537" alt="image" src="https://github.com/user-attachments/assets/328065de-fdfd-47e5-a7c8-48a360ba3a16" />

## Datos sensibles o roles expuestos en texto plano

El payload expone, sin ningún tipo de cifrado, los siguientes datos:

| Campo | Tipo de dato | Por qué es sensible |
|---|---|---|
| `email` | Dato personal (PII) | Identifica a la persona directamente |
| `fullname` | Dato personal (PII) | Nombre completo del usuario |
| `role_id` | Control de acceso | Define el rol/permisos del usuario dentro del sistema |
| `is_superadmin` | Control de acceso | Indica si la cuenta tiene privilegios de administrador |
| `plan_id`, `pagado`, `fecha_pago` | Datos de facturación | Información comercial/de suscripción del usuario |
| `confirmation_email_uuid` | Identificador interno | Si algún endpoint de la app usa este UUID para confirmar el email sin otra validación, un atacante que lo obtenga podría intentar reutilizarlo |
| `iat`, `exp` | Metadatos de sesión | Marca cuándo fue emitido y cuándo expira el token |

El campo más crítico desde el punto de vista de control de acceso es **`is_superadmin`** / **`role_id`**: si el backend confía en estos valores leídos directamente del payload del token (en vez de validarlos contra la base de datos en cada request), y además el token pudiera ser modificado (por ejemplo, si el servidor no verificara correctamente la firma, como en el ataque de algoritmo `none` del Objetivo 4), un atacante podría intentar escalar privilegios cambiando `role_id` o `is_superadmin` en un token falsificado.

## Conclusión

Un JWT **no es un mecanismo de confidencialidad**, es un mecanismo de **integridad**: protege el contenido contra modificaciones (mediante la firma), pero no contra la lectura. Cualquiera que intercepte el token —por ejemplo, con las DevTools del navegador, como se hizo en esta prueba— puede leer absolutamente todo el payload sin necesitar ninguna clave ni contraseña. Por esta razón, **nunca se debe guardar en el payload de un JWT información verdaderamente secreta** (contraseñas, tokens de terceros, datos financieros completos, etc.), solo datos que el cliente ya tiene derecho a conocer sobre sí mismo.
