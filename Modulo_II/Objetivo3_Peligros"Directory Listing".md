# Objetivo 3: Peligros del "Directory Listing"

## Qué es el Directory Listing?

Cuando un navegador pide una URL que apunta a un **directorio**, el servidor web (Apache, Nginx, IIS, etc.) tiene que decidir qué devolver. Normalmente busca un **archivo de índice** dentro de esa carpeta (`index.html`, `index.php`, `default.aspx`, etc.) y lo sirve automáticamente — por eso, al entrar a la raíz de casi cualquier sitio, ves la página principal sin tener que escribir `index.html` en la URL.

El problema aparece cuando:
1. **Esa carpeta no contiene ningún archivo de índice**, y
2. La directiva de configuración del servidor que controla esto (en Apache: `Options +Indexes`; en Nginx: `autoindex on;`) está **habilitada**.

En ese caso, en vez de devolver un error 403/404, el servidor genera automáticamente una página HTML que **lista todos los archivos y subcarpetas** que existen físicamente dentro de ese directorio — nombre, tamaño, fecha de modificación — como si fuera un explorador de archivos, accesible públicamente por cualquiera que conozca o adivine esa URL.

## Por qué esto es un error de configuración y no una funcionalidad

En la enorme mayoría de los casos, esto **no es una funcionalidad que el desarrollador quiso implementar** — es un comportamiento por defecto del servidor web que queda activo porque nadie lo desactivó explícitamente. El desarrollador espera que esas carpetas (ej. `/uploads/`, `/backups/`, `/assets/`, `/logs/`) solo sean accedidas indirectamente, mediante links específicos a archivos puntuales que la aplicación genera — nunca imaginó que alguien iba a navegar directamente a la carpeta y ver **todo** su contenido.

## Qué tipo de archivos sensibles pueden quedar expuestos

Dependiendo de la carpeta, el Directory Listing puede exponer:

- **Backups de la aplicación o de la base de datos:** archivos como `backup.sql`, `db_dump_2026.zip`, `site_backup.tar.gz` — potencialmente con credenciales, datos de usuarios, estructura completa de la base de datos.
- **Archivos de configuración:** `.env`, `config.php.bak`, `wp-config.php.old` — muchas veces contienen contraseñas de base de datos, claves de API, secretos de sesión (como el que vimos en el Objetivo 2 con el JWT firmado con HS256 — si la clave secreta quedara en un archivo de config expuesto, cualquiera podría firmar tokens falsos).
- **Código fuente sin procesar:** archivos `.php`, `.py`, `.java` que, si se acceden directamente en vez de ser ejecutados por el servidor, pueden revelar la lógica de negocio completa, credenciales hardcodeadas, lógica de validación (información que facilita directamente ataques de lógica de negocio, como los del Objetivo 1 de este módulo).
- **Archivos subidos por usuarios sin control de acceso:** documentos, fotos privadas, contratos — si la carpeta de uploads no tiene control de acceso por archivo (solo "seguridad por oscuridad", asumiendo que nadie va a adivinar el nombre del archivo), el listado del directorio elimina esa única barrera.
- **Logs del servidor o de la aplicación:** `error.log`, `access.log` — pueden contener stack traces (ver Objetivo 4), rutas internas, IPs, y en algunos casos hasta parámetros sensibles si quedaron logueados por error (tokens, contraseñas enviadas por GET).
- **Archivos de control de versiones expuestos por accidente:** una carpeta `.git/` olvidada en el servidor de producción, por ejemplo, permite a un atacante reconstruir el historial completo del código fuente del repositorio.

## Por qué esto facilita un ataque

El Directory Listing elimina la necesidad de **adivinar** nombres de archivo (lo que normalmente requeriría fuerza bruta de rutas, con herramientas como `gobuster` o `ffuf`, y podría no encontrar nada si el nombre es poco común). En cambio, el servidor le entrega al atacante, gratis y en una sola petición, el **índice completo** de todo lo que hay en esa carpeta — convirtiendo el reconocimiento de "a ciegas, probando nombres" a "lectura directa del mapa completo".

## Cómo se mitiga

- **Desactivar explícitamente el listado de directorios** a nivel de configuración del servidor web:
  - En Apache: `Options -Indexes` (en el `.htaccess` o en el bloque `<Directory>` de la configuración del vhost).
  - En Nginx: asegurarse de que `autoindex off;` (que es el valor por defecto, pero puede haber sido activado accidentalmente).
- **Colocar siempre un archivo de índice vacío o de redirección** (`index.html` vacío, o uno que redirija al home) en cada carpeta pública como segunda capa de seguridad, por si la directiva anterior fallara.
- **No depender de "seguridad por oscuridad":** cualquier archivo verdaderamente sensible (backups, configs, uploads privados) no debería estar accesible desde una ruta pública del servidor web en absoluto — debería estar fuera del `document root`, o protegido con autenticación/autorización a nivel de aplicación, independientemente de si el listado de directorio está activo o no.
- **Auditorías periódicas:** escanear el propio sitio en busca de carpetas con listados accidentalmente habilitados, y revisar que no haya archivos de backup, `.git`, `.env` o similares dentro del `document root` servido públicamente.

## Conclusión

El Directory Listing es un ejemplo claro de cómo un **error de configuración**, no una vulnerabilidad de código, puede exponer información crítica: convierte cualquier carpeta sin archivo de índice en un listado público de todo su contenido, eliminando la necesidad de que el atacante adivine nombres de archivo y entregándole directamente un mapa de backups, configuraciones, código fuente y archivos de usuarios que nunca debieron ser accesibles desde la web.
