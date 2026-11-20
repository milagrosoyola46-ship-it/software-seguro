# Plantilla de Reporte de Pentesting

> Plantilla reutilizable para reportar hallazgos de pentesting.

---

## Datos del Reporte

| Campo | Detalle |
|---|---|
| **Proyecto** | *(nombre del proyecto)* |
| **Tipo de prueba** | Caja negra / Caja gris / Caja blanca |
| **Fecha de inicio** | dd/mm/aaaa |
| **Fecha de fin** | dd/mm/aaaa |
| **Autor** | *(mi nombre)* |
| **Versión del reporte** | v1.0 |
| **Alcance** | *(URLs, rangos de IP o funcionalidades incluidas)* |
| **Fuera de alcance** | *(lo que no se probó)* |

---

## Resumen Ejecutivo

> **Audiencia:** directivos, gerentes, personas no técnicas. 

**Objetivo de la evaluación:** qué se evaluó y por qué

**Resultado general:** veredicto general

**Resumen de hallazgos por severidad:**

| Severidad | Cantidad |
|---|---|
| 🔴 Crítica | 0 |
| 🟠 Alta | 0 |
| 🟡 Media | 0 |
| 🔵 Baja | 0 |
| ⚪ Informativa | 0 |

**Riesgo de negocio:** qué pasaría si estas vulnerabilidades son explotadas

**Recomendación general:** 

---

## Detalles Técnicos

> **Audiencia:** equipo técnico / desarrollo. Acá se documenta cada hallazgo en profundidad.

Por cada vulnerabilidad encontrada, se repite esta estructura:

### [ID-01] Título del hallazgo (ej: IDOR en endpoint de descarga de fotos)

- **Severidad:** Crítica / Alta / Media / Baja / Informativa
- **Categoría (OWASP):** ej. A01:2021 – Broken Access Control
- **Endpoint / componente afectado:** `GET /fotos/{id}/raw`
- **Descripción técnica:** qué hace el endpoint, cuál es la falla exacta y por qué ocurre (causa raíz).
- **Impacto técnico:** qué puede hacer un atacante concretamente (ej: leer fotos privadas de cualquier usuario cambiando un ID numérico).
- **Condiciones necesarias:** qué necesita el atacante para explotarlo (ej: una cuenta válida, pero sin ningún privilegio especial).

---

## Métricas CVSS

> Cuantificar la severidad de forma objetiva y comparable entre hallazgos, usando el estándar de la industria (CVSS v3.1).

| ID | Hallazgo | Vector CVSS | Score | Severidad |
|---|---|---|---|---|
| ID-01 | *(nombre)* | `CVSS:3.1/AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:N/A:N` | 7.1 | Alta |

**Desglose del vector (ejemplo):**
- **AV (Attack Vector):** N = Network — explotable remotamente
- **AC (Attack Complexity):** L = Low — no requiere condiciones especiales
- **PR (Privileges Required):** L = Low — requiere una cuenta de usuario normal
- **UI (User Interaction):** N = None — no requiere que la víctima haga nada
- **S (Scope):** U = Unchanged
- **C/I/A (Confidentiality/Integrity/Availability):** impacto en cada dimensión

> Usar una calculadora oficial (https://www.first.org/cvss/calculator/3.1) para generar el vector real de cada hallazgo.

---

## Prueba de Concepto (PoC) — Paso a Paso

> **Audiencia:** quien va a reproducir y validar el hallazgo. Debe ser reproducible por un tercero sin ambigüedad.

### PoC – [ID-01]

**Pre-requisitos:**
- Cuenta de usuario válida
- Proxy Burp Suite configurado en el navegador

**Pasos:**
1. Iniciar sesión con la cuenta `usuario_a`.
2. Subir una foto propia y marcarla como privada.
3. Interceptar con Burp la petición `GET /fotos/{id_propio}/raw` y confirmar acceso (200 OK).
4. Modificar manualmente el `{id}` por el de una foto ajena marcada como privada (ej: `/fotos/31/raw`).
5. Reenviar la petición.

**Evidencia:**

`![Captura: petición original](ruta/a/imagen1.png)`

`![Captura: petición modificada / respuesta obtenida](ruta/a/imagen2.png)`

**Resultado observado:** descripción de qué devolvió el servidor (código de estado, contenido de la respuesta) y por qué confirma la vulnerabilidad.

---

## Recomendaciones de Remediación

| ID | Hallazgo | Recomendación | Prioridad | Esfuerzo estimado |
|---|---|---|---|---|
| ID-01 | *(nombre)* | Validar en el backend que el `id` solicitado pertenece al usuario autenticado antes de servir el recurso (control de acceso a nivel de objeto) | Alta | Bajo |

**Recomendaciones generales (aplicables a todo el sistema):**
- Implementar controles de autorización a nivel de objeto (no solo autenticación) en todos los endpoints que reciban un identificador.
- Agregar pruebas automatizadas de control de acceso en el pipeline de CI/CD.
- Realizar una nueva ronda de verificación (retest) una vez aplicadas las correcciones.

---
## Explicación de la plantilla

Esta plantilla está pensada para que un mismo reporte sirva a distintos lectores sin que ninguno tenga que leer de más:

Resumen Ejecutivo: le da a un directivo o gerente, en segundos, el dato que necesita para tomar una decisión de negocio sin exponerlo a lenguaje técnica que no va a entender ni necesita.
Detalles Técnicos: le da al equipo de desarrollo el contexto completo de la falla para que puedan entenderla sin tener que pedir una reunión aparte.
Métricas CVSS: saca la subjetividad de la discusión de "qué es urgente": al usar un estándar reconocido, se puede priorizar objetivamente y comparar hallazgos entre sí o entre distintos reportes/auditorías.
PoC paso a paso: elimina la ambigüedad — cualquiera en el equipo puede reproducir el problema exactamente como lo hizo el pentester, lo cual es clave para validar la corrección después (retest).
Recomendaciones de Remediación: convierte el hallazgo en una acción concreta y accionable, con prioridad y esfuerzo, para que pueda entrar directo a un backlog de tickets sin que desarrollo tenga que "traducir" el hallazgo a una tarea.

Valor para quien lee un ticket armado con esta plantilla: cada rol (directivo, desarrollador, QA, otro pentester que haga el retest) encuentra justo la sección que necesita, en el nivel de detalle que le sirve, sin ruido — lo que acorta el tiempo entre "se encontró una vulnerabilidad" y "se corrigió y se verificó"
