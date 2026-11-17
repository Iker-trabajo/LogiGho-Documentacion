# ADR-001 — Identidad verificada en la Lambda y catálogo de experiencias en el servidor

**Autor:** Iker Acevedo  
**Fecha:** 2026-11-10  
**Estado:** Propuesta (pasa a Aceptada cuando Iker apruebe la salida a producción)

---

## Contexto

El Sistema de Experiencias necesita guardar la experiencia visual de cada usuario y decir qué experiencias puede usar. Hay tres decisiones que no son obvias:

1. **De dónde sale la identidad.** El authorizer Cognito del REST valida el token, pero en PreProd ya hubo rutas del mismo REST creadas con autorización `NONE` (métricas) y el authorizer de PreProd apunta al pool de la cuenta de producción. Confiar solo en que el gateway filtre es frágil.
2. **Quién decide la disponibilidad** de una experiencia (`borrador`, `beta`, `general`, `retirada`): el front no puede, porque cualquiera puede alterar el código del navegador.
3. **Dónde vive la configuración** del catálogo y cómo se cambia con seguridad.

También hubo que decidir cómo publicar las rutas en el REST compartido sin romper a los demás dominios.

---

## Opciones consideradas

### Opción A — Identidad solo del authorizer (claims del gateway)

La Lambda lee `requestContext.authorizer.claims.sub`.

**Pros:** menos código, sin descargar JWKS.  
**Contras:** si una ruta queda con autorización `NONE` (ya pasó en PreProd) cualquiera suplanta a cualquiera; sin claims la Lambda no sabría que no está autenticado.

### Opción B — Authorizer del gateway + verificación de firma del JWT en la Lambda (elegida)

La Lambda verifica RS256, `iss`, `exp`, `nbf`, `token_use` y app client contra el JWKS del user pool, y toma el `sub` solo de ese token. El authorizer sigue puesto en las rutas.

**Pros:** defensa en profundidad; es el patrón ya usado en métricas e inventario; falla cerrada si falta configuración.  
**Contras:** código duplicado del verificador (mejora pendiente: librería común); una descarga del JWKS por contenedor.

### Opción C — Catálogo en una colección de MongoDB administrable desde una pantalla

**Pros:** cambiar el catálogo sin desplegar.  
**Contras:** una escritura de administración más que proteger, una pantalla más, y cada `GET` dependería de otra consulta. Excesivo para empezar.

---

## Decisión

1. **Identidad:** opción B. El `sub` sale únicamente del JWT verificado dentro de la Lambda y se valida con un formato estricto. La ruta es `/preferencias/mia` y no acepta identificadores.
2. **Disponibilidad:** la decide el servidor. `general` para todos; `beta` para quienes tengan el rol `Experiencia` (variable `ROLES_BETA`) o `Desarrollador`; `borrador` solo `Desarrollador` (`ROLES_BORRADOR`); `retirada` para nadie. Los roles se leen de `Users` y **nunca** se devuelven ni se aceptan del cliente. Cuando una experiencia pasa a `general`, la ven todos los usuarios (el front sigue mostrando a cada uno solo sus módulos asignados).
3. **Catálogo:** *(sustituida por [ADR-002](ADR-002-catalogo-en-base-de-datos-y-aot.md): el catálogo ahora vive en la colección `Experiencias`)* variables de entorno gestionadas por **Terraform** (`EXPERIENCIAS`, `TEMAS`, `ROLES_*`, `ORIGEN_PERMITIDO`). Cambiarlo es un cambio revisado con `plan` y reversible volviendo a `clasica:general`. La Lambda **falla cerrada** si la configuración es inválida (`clasica:general` siempre presente).
4. **Publicación:** el stack `core` crea sus rutas en el REST existente y publica `ids_para_despliegue`; el dueño del REST de PreProd (`environments/preprod/datos`) los suma al trigger de su deployment (el mismo mecanismo de métricas y autenticación). En producción el deployment lo publica Iker con el CLI. Se descartó `create-deployment` desde el pipeline con `ignore_changes` porque rompía la regla de un solo dueño del stage.
5. **CORS:** el preflight `OPTIONS` lo responde la Lambda (no un MOCK del gateway) para devolver el origen exacto entre una lista cerrada; nunca `*`.
6. **Base de datos:** se usa la cadena actual (decisión de Iker 2026-11-10). La Lambda solo toca `PreferenciasUsuario` (lectura/escritura) y `Users` (lectura con proyección de `roles`). El usuario de base de datos con permisos mínimos queda como mejora.

---

## Consecuencias

**Positivas**
- Un token falso o una ruta mal configurada no permiten leer ni escribir preferencias ajenas.
- Un front manipulado no puede activar una experiencia que no le corresponde (422).
- Revertir todo es un cambio de una línea (`clasica:general`) y el front tolera la ausencia de la API.

**Negativas / riesgos aceptados**
- Código del verificador JWT duplicado en tres Lambdas (mejora: librería compartida).
- La Lambda tiene acceso a toda la base por la cadena de conexión compartida (mejora: usuario mínimo).
- Un `401` del authorizer no lleva cabeceras CORS (comportamiento del REST compartido).

---

## Relacionado

- [API de Preferencias](../../../api/preferencias.md) · [Ficha de la Lambda](index.md)
- [Seguridad por diseño](../../../frontend/experiencias/seguridad-por-diseno.md) · [ADR-001 del Sistema de Experiencias](../../../frontend/experiencias/adr/ADR-001-sistema-experiencias.md)

---

## Autor: Iker Acevedo
Fecha creacion: 2026-11-10  
Estado: propuesta
