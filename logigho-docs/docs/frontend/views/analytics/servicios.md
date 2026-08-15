---
autor: Iker Acevedo Vargas
fecha_creacion: 2026-08-14
ultima_actualizacion: 2026-08-14
estado: desarrollo
---

# Servicios: Analytics

**Ubicación:** `src/app/views/analytics/services`
**Scope:** compartidos por las 3 páginas de la sección — ninguno sale de `views/analytics` (ver regla de organización en el [overview](overview.md#arquitectura-de-carpetas))

---

## `ReportesAnalyticsService`

**Ubicación:** `services/reportes-analytics.service.ts`

CRUD de Mongo + subida/lectura de S3 + compresión + verificación de integridad. Es el único punto de entrada a los datos del módulo — ninguna vista llama a `ConsumoGenericoService`/`PutObjectService`/`GetObjectService` directamente.

### `listarReportes(): Promise<ReporteAnalytics[]>`

Trae todos los documentos de la colección. La respuesta del backend viene gzip-comprimida en base64 (mismo patrón que otros `metodoGenerico`); se descomprime con `DecompressionService.decompressGzip()`.

### `publicarVersion(html, nombreArchivo, datos, reporteExistente): Promise<void>`

Si `reporteExistente` es `null`, crea el documento; si no, hace `push` a `Versiones[]` y mueve `VersionActiva` — versionado real, ninguna key de S3 se sobrescribe. Internamente: gzipea el HTML (`pako`), calcula SHA-256, sube a S3, inserta/actualiza en Mongo.

### `actualizarMetadatos(reporte, datos): Promise<void>`

Edita nombre/descripción/categoría/estado/roles **sin** tocar S3 ni crear una versión nueva.

### `cambiarEstado(reporte, estado): Promise<void>`

`ACTIVO` ⇄ `ARCHIVADO`. Nunca borra nada — solo cambia el campo `Estado`.

### `restaurarVersion(reporte, versionId): Promise<void>`

Rollback: mueve `VersionActiva` a una versión anterior, sin tocar el array `Versiones`.

### `obtenerHtmlVersion(reporte, versionId?): Promise<string>`

Descarga de S3, descomprime, y **recalcula el SHA-256** contra el guardado en Mongo — si no coincide, lanza error en vez de renderizar. Detecta manipulación del objeto en S3 hecha por fuera de la plataforma.

| Parámetro | Tipo | Descripción |
| --- | --- | --- |
| `reporte` | `ReporteAnalytics` | El reporte dueño de la versión |
| `versionId` | `string` (opcional) | Si se omite, usa `reporte.VersionActiva` |

**Constantes internas:**

```ts
const BUCKET_REPORTES = 'logigho-plantillas';   // temporal — ver overview
const PREFIJO_REPORTES = 'reportes-analytics';
```

---

## `ReporteSanitizerService`

**Ubicación:** `services/reporte-sanitizer.service.ts`

Capa 2 (CSP) y Capa 3 (validación en subida) del modelo de seguridad. Ver el detalle completo en el [overview](overview.md#modelo-de-seguridad).

### `validar(html, nombreArchivo): ResultadoValidacion`

Corre una lista de patrones prohibidos (`RECURSO_EXTERNO`, `SALIDA_DE_RED`, `ETIQUETA_PROHIBIDA`, `ACCESO_AL_ANFITRION`, `NAVEGACION`, `META_REFRESH`, `IMPORT_DINAMICO`, `IMPORT_CSS_EXTERNO`) línea por línea. Devuelve `{ valido, hallazgos[] }`, cada hallazgo con línea y extracto exacto.

> No es la defensa real — las capas 1 y 2 bloquean todo aunque un patrón se les escape (ofuscación tipo `window['fe'+'tch']`, documentado y cubierto por test en el spec del servicio). Es feedback temprano para el autor.

### `endurecerHtml(html): string`

Inserta la CSP como primer hijo de `<head>` (crea `<head>` si no existe) y elimina cualquier `<base>` del reporte.

### `obtenerReglasExpuestas(): ReglaExpuesta[]`

Expone la tabla de reglas prohibidas (sin el patrón regex) para que `ReglasAutoresReporteComponent` la renderice — nunca se desincroniza de la validación real porque es la misma fuente.

---

## `formato.util.ts`

**Ubicación:** `services/formato.util.ts` — funciones puras, no un `@Injectable`

| Función | Firma | Ejemplo |
| --- | --- | --- |
| `formatearFechaCorta` | `(fechaIso: string) => string` | `"14 Ago, 2026"` |
| `formatearFechaLarga` | `(fechaIso: string) => string` | `"14 Ago 2026, 14:30"` |
| `formatearTamano` | `(bytes: number) => string` | `"1.6 MB"` |

Sin registrar un `LOCALE_ID` global de Angular — eso afectaría a toda la plataforma, no solo a este módulo.

---

## `roles.util.ts`

**Ubicación:** `services/roles.util.ts` — funciones puras, no un `@Injectable`

| Función | Firma | Descripción |
| --- | --- | --- |
| `obtenerRolesUsuario` | `() => string[]` | Lee `sessionStorage.roles_asignados` (mismo mecanismo que el sidebar) |
| `puedeGestionarReportes` | `() => boolean` | `true` si el usuario tiene rol `Jefe Datos` o `Desarrollador` |

> Solo control de UI (mostrar/ocultar el botón "Gestionar ETL") — no reemplaza autorización de backend, que hoy no existe para este módulo.

---

## Historial de cambios

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-08-14 | Iker Acevedo Vargas | Documentación inicial de los 4 archivos de `services/` |
