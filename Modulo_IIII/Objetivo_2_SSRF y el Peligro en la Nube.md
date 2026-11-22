# Objetivo 2: SSRF y el Peligro en la Nube (AWS/GCP)

## Qué es la dirección 169.254.169.254

`169.254.169.254` es una dirección IP especial reservada (dentro del rango *link-local*, APIPA) que **todos los grandes proveedores de nube** (AWS, GCP, Azure, Alibaba Cloud, etc.) usan para exponer el **servicio de metadatos de la instancia** (Instance Metadata Service, IMDS). Cualquier máquina virtual corriendo en esa nube puede consultar esa dirección **desde adentro** de la propia instancia, sin autenticación, para obtener información sobre sí misma: su ID, su región, sus tags, su configuración de red, y — lo más crítico — **credenciales temporales de IAM** asociadas al rol que tiene asignada esa instancia.

Esta dirección **no es enrutable desde internet** — solo responde a peticiones que se originan *dentro* de la red interna de la nube, generalmente desde el propio servidor. Por eso, en condiciones normales, es inofensiva: un atacante externo no puede alcanzarla directamente.

## Por qué un SSRF la convierte en catastrófica

Una vulnerabilidad de **SSRF (Server-Side Request Forgery)** ocurre cuando una aplicación tiene una funcionalidad que hace peticiones HTTP a una URL que el atacante puede controlar total o parcialmente (por ejemplo, una función de "importar imagen desde URL", "generar PDF de esta página", o un "diagnóstico de dispositivo remoto" como vimos en un laboratorio anterior).

Si el atacante logra que esa petición apunte a:

```
http://169.254.169.254/latest/meta-data/iam/security-credentials/<nombre-del-rol>
```

**el servidor vulnerable hace la petición por sí mismo**, desde adentro de la red de la nube, donde esa dirección sí es alcanzable — el atacante nunca necesita acceso directo a la red interna, usa al propio servidor como "proxy" involuntario para llegar ahí. El servidor obtiene la respuesta (las credenciales) y, si la aplicación es además vulnerable de forma que refleja o devuelve el contenido de esa respuesta al cliente (lo cual es extremadamente común en funciones de "diagnóstico" o "preview de URL"), el atacante termina viendo esas credenciales directamente en su propia pantalla.

## Qué información altamente sensible espera extraer el atacante

En AWS específicamente, el flujo típico de ataque es:

1. Primero, listar qué rol IAM tiene asignado la instancia:
   ```
   GET http://169.254.169.254/latest/meta-data/iam/security-credentials/
   ```
   (devuelve el nombre del rol, ej: `prod-app-server-role`)

2. Después, pedir las credenciales de ese rol específico:
   ```
   GET http://169.254.169.254/latest/meta-data/iam/security-credentials/prod-app-server-role
   ```
   Esto devuelve un JSON con:
   - **`AccessKeyId`**
   - **`SecretAccessKey`**
   - **`Token`** (session token)
   - Fecha de expiración

Estas son **credenciales de AWS completamente funcionales**, temporales pero válidas, que el atacante puede cargar directamente en su propia terminal (`aws configure` o variables de entorno `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_SESSION_TOKEN`) y usar para autenticarse contra la API de AWS **con los mismos permisos que tiene la aplicación legítima** — sin haber roto ninguna contraseña ni haber hackeado ninguna cuenta de usuario.

Dependiendo de qué permisos tenga el rol IAM asignado a esa instancia (que en muchas configuraciones reales son más amplios de lo necesario — "over-permissioned roles"), esto puede escalar a:
- Leer o modificar datos en buckets de **S3** (incluyendo backups, datos de clientes).
- Leer secretos guardados en **AWS Secrets Manager** o **Systems Manager Parameter Store** (contraseñas de bases de datos, claves de API de terceros).
- Enumerar o modificar otras instancias EC2, bases de datos RDS, funciones Lambda.
- En el peor de los casos, si el rol tiene permisos administrativos amplios, **tomar control de toda la cuenta de AWS**.

(En Google Cloud Platform el concepto es análogo: la misma IP `169.254.169.254` expone el metadata server de GCP, con endpoint típico `http://169.254.169.254/computeMetadata/v1/instance/service-accounts/default/token`, que entrega un token OAuth de la cuenta de servicio asociada a la VM — con el mismo tipo de riesgo.)

## Por qué esto es "catastrófico" y no solo "grave"

A diferencia de un SSRF que solo permite leer un archivo interno puntual o escanear puertos internos, llegar al metadata service significa que el atacante obtiene **credenciales válidas de la nube en sí misma** — pasa de "comprometí esta aplicación web" a "tengo acceso con una identidad legítima a toda la infraestructura de nube que esa identidad puede tocar". Esto puede saltar completamente los límites de la aplicación vulnerable y afectar otros sistemas, bases de datos y servicios que ni siquiera están relacionados con la app original.

## Mitigación

- **Usar IMDSv2 en AWS** (que requiere un token de sesión obtenido mediante una petición `PUT` previa con un header especial), lo que hace mucho más difícil explotarlo mediante un SSRF simple basado en `GET`, ya que la mayoría de los SSRF no pueden forzar fácilmente una petición `PUT` con headers custom.
- **Asignar permisos mínimos (principio de menor privilegio)** a los roles IAM de las instancias, para que incluso si las credenciales son robadas, el daño posible esté acotado.
- **Validar y restringir estrictamente** cualquier funcionalidad del lado del servidor que acepte una URL proporcionada por el usuario (whitelist de dominios permitidos, bloqueo explícito de rangos de IP internas/link-local, incluyendo `169.254.169.254` y notaciones alternativas como decimal/octal/IPv6 que pueden evadir filtros mal implementados).
- **Segmentación de red:** cuando sea posible, restringir el acceso al metadata service a nivel de firewall/red para servicios que no lo necesiten.

## Conclusión

En un entorno cloud, un SSRF deja de ser "solo" una forma de alcanzar servicios internos y se convierte en una vía directa para robar las credenciales de identidad de la propia infraestructura — porque la dirección `169.254.169.254` existe precisamente para entregarle a cualquier proceso dentro de la instancia acceso sin contraseña a esas credenciales, bajo la premisa (rota por el SSRF) de que solo el software legítimo de esa máquina podría llegar a pedirlas.
