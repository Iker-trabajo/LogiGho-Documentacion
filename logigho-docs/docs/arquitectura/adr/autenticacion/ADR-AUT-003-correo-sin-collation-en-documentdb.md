---

## Autor: Iker Acevedo
Fecha creacion: 2026-11-20
Estado: aceptada

# ADR-AUT-003 — El correo se compara con regex, no con Collation

**Autor:** Iker Acevedo (decisión) · redactado con Claude
**Fecha:** 2026-11-20
**Estado:** Aceptada

---

## Contexto

Desde el despliegue de autenticación en producción (2026-11-18), `POST /createUser` respondía `500 ERROR_INTERNO` y el administrador no podía crear usuarios. El log de la lambda decía `[ERROR] crear-usuario excepcion=MongoCommandException`: Mongo rechazaba un comando, no se caía la conexión.

La comprobación de correo duplicado de `createUser` usaba `CountDocuments` con `Collation("en", Secondary)` para ignorar mayúsculas. Era el único uso de `Collation` en el repo de lambdas. Producción corre sobre Amazon DocumentDB, cuyo soporte de collation es limitado, y PreProd corre sobre Atlas, que sí la acepta. **Causa probable (hipótesis)**; se confirma al desplegar el arreglo (ver Verificación).

## Opciones consideradas

### Opción A — Regex anclada con opción `i` (elegida)

`^<correo escapado>$` con `i`. Es la técnica que ya usa `UsuariosMongo` en el registro, que funciona en producción.

**Pros:** funciona igual en Atlas y DocumentDB; no cambia los datos; el escape evita que `+` o `.` amplíen la búsqueda.
**Contras:** no usa un índice de forma eficiente (la regex insensible recorre la colección). Aceptable con ~400 usuarios.

### Opción B — Guardar y buscar el correo en minúsculas

**Pros:** índice exacto.
**Contras:** exige migrar los usuarios existentes y tocar todas las rutas que escriben `email`. Descartada por el riesgo en producción.

### Opción C — Mantener Collation y subir de versión DocumentDB

**Contras:** depende de infraestructura y no desbloquea al administrador hoy.

## Decisión

Opción A. Además, un rechazo de Mongo ahora se registra con `codigo` y `nombre` del error (sin el mensaje, que puede traer correos), para diagnosticar la próxima falla sin adivinar.

## Consecuencias

- Regla para cualquier lambda nueva: **no usar `Collation` contra producción**. Probar siempre contra DocumentDB, no solo contra Atlas de PreProd.
- Log esperado: `[ERROR] crear-usuario excepcion=MongoCommandException codigo=<n> nombre=<CodeName>`.

## Verificación

Tras el despliegue: crear un usuario desde Administración › Usuarios (200) y repetir el correo cambiando mayúsculas (409 `USUARIO_DUPLICADO`). Si siguiera el 500, el nuevo log trae el código exacto.
