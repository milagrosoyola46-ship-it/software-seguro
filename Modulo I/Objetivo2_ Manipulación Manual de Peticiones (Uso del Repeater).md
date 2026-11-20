# Objetivo 2: Manipulación Manual de Peticiones (Uso del Repeater)

**Laboratorio:** Gran Rifa 2019

## Interceptación de la petición original

Con Burp Suite configurado como proxy local y el certificado de la CA instalado en el navegador, se accedió al formulario de login del laboratorio y se envió un intento de autenticación con credenciales de prueba. Burp interceptó la petición `POST /login/?next=/` antes de que llegara al servidor.

<img width="951" height="1015" alt="image" src="https://github.com/user-attachments/assets/aa6cf1a1-d6cf-4054-8357-7961ec765b00" />

La petición original incluye, entre otros:
- **Endpoint:** `POST /login/?next=/`
- **Host**
- **Cookie** 
- **User-Agent original:** navegador real
- **Body** parámetros de usuario y clave del formulario

Esta petición se envió al módulo Repeater, dejando pasar la original con Forward para no interrumpir la navegación.

<img width="587" height="741" alt="image" src="https://github.com/user-attachments/assets/d7649250-c65c-4953-b35a-ec84341f58f0" />

## Modificación manual del parámetro

Se modificó manualmente el header **User-Agent**, reemplazando el valor del navegador real por otro:

```
User-Agent: TestBot/1.0 (Laboratorio-Software-Seguro)
```

Este cambio simula una petición generada por una herramienta automatizada en lugar de un navegador real, para verificar si el servidor aplica algún control sobre este header.

<img width="536" height="716" alt="image" src="https://github.com/user-attachments/assets/c9d641d7-44fa-4c49-b7b9-615a88e93139" />

## Envío y análisis de la respuesta

Al enviar la petición modificada, el servidor respondió:

<img width="582" height="757" alt="image" src="https://github.com/user-attachments/assets/516c8535-2e1a-4eee-a7ca-69f9dba5718c" />

El servidor devolvió un `200 OK` con el HTML normal de la página de login, el mismo contenido que se obtiene con un User-Agent de navegador real, sin ningún bloqueo ni respuesta diferenciada.

## Conclusión

- El servidor **no valida ni restringe el header `User-Agent`**: acepta la petición sin importar si el valor corresponde a un navegador real o a uno inventado.
- Esto implica que un atacante podría automatizar peticiones (fuerza bruta, scraping, fuzzing) sin que el User-Agent sea una barrera.
