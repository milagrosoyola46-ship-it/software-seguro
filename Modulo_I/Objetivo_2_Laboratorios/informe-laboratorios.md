# Informes de Laboratorios — Módulo I

## Laboratorio: Gran Rifa 2019

El desafío planteaba una situación en la que John Backus, se había anotado para comprar una rifa pero todavía no la había pagado y no tenía intención de hacerlo. El objetivo era encontrar la forma de que esa rifa quedara marcada como pagada sin que un pago real hubiera ocurrido.

Para resolverlo, lo primero que hice fue iniciar sesión en la aplicación con el usuario y contraseña. Una vez adentro, abrí las herramientas de desarrollador del navegador con F12 y me fui a la pestaña Network, que muestra todas las peticiones.

Al hacer click en el botón "Editar" de alguna de las filas de la tabla de rifas, pude ver que la aplicación mandaba una petición de tipo POST hacia un endpoint, y que esa petición llevaba en su cuerpo un campo llamado `esta_pago`, con el valor `false` para la rifa de John Backus. Entonces ese mismo campo podía modificarse directamente desde ahí, sin que lo pague realmente.

Entonces tomé esa misma petición, cambié el valor de `esta_pago` de `false` a `true`, y la envié nuevamente desde las herramientas de desarrollador. El servidor respondió con un mensaje `{"estado": "OK"}`, indicando que el cambio se había echo correctamente. Al recargar la pagina, la fila correspondiente a John Backus ahora mostraba "Está pago: Sí".

## Laboratorio: Turnero



## Laboratorio: Ventas


## Laboratorio: Presupuesto
