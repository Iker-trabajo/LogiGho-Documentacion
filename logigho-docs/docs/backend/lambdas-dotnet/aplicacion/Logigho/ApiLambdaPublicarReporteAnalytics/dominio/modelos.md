## Autor: Iker Acevedo

Fecha creacion: 2026-09-21

Estado: desarrollo

## Dominio: Entidades, Enums y Validación

**Ubicación:** `Dominio/Entidades/`, `Dominio/Enums/`, `Dominio/Validacion/`

Capa sin dependencias hacia Mongo, S3, AWS ni JSON — solo el modelo de negocio.

---

## `ReporteAnalytics` — el documento final

Colección `ReportesAnalytics`. **Mismo esquema que usa hoy Angular** — se verificó
campo por campo contra un export real de producción, incluido el campo `Encoding` de
cada versión.

| Campo | Tipo | Notas |
| ----- | ---- | ----- |
| `_id` | `string?` | ObjectId de Mongo |
| `Id` | `string` | Campo legado, siempre `""` |
| `ReporteId` | `string` | Identificador estable del reporte, `Guid` |
| `Nombre`, `Descripcion`, `Categoria` | `string` | Metadatos editables |
| `Estado` | `string` | `ACTIVO` \| `BORRADOR` \| `ARCHIVADO` |
| `RolesPermitidos` | `string[]` | Quién puede ver el reporte en la plataforma |
| `VersionActiva` | `string` | `VersionId` de la versión que se muestra por defecto |
| `Versiones` | `VersionReporte[]` | **Nunca se reemplaza** — cada publicación agrega, no sobreescribe |
| `FechaCreacion`, `FechaActualizacion` | `string` | ISO UTC |
| `CreadoPor`, `ActualizadoPor` | `string` | Email/username, nunca el `sub` |

---

## `VersionReporte` — una versión inmutable dentro de `Versiones[]`

| Campo | Tipo | Notas |
| ----- | ---- | ----- |
| `VersionId` | `string` | `Guid` propio de esa versión |
| `S3Bucket`, `S3Key` | `string` | `S3Key` = `reportes-analytics/{reporteId}/{versionId}.html.gz` |
| `NombreArchivo` | `string` | Nombre original declarado por el cliente |
| `TamanoBytes` | `long` | Tamaño del HTML crudo (UTF-8) |
| `TamanoComprimidoBytes` | `long` | Tamaño del gzip, **antes** de codificar a base64 |
| `Encoding` | `string` | Siempre `"gzip+base64"` |
| `Sha256` | `string` | Hash lowercase de los bytes UTF-8 crudos (no del gzip) |
| `Autor` | `string` | Email/username de quien publicó esa versión puntual |
| `FechaSubida` | `string` | ISO UTC |
| `Notas` | `string` | Texto libre del formulario de publicación |

**El contenido en S3 nunca es HTML plano ni gzip binario — es el texto base64 de un
gzip**, aunque la extensión de la key sea `.html.gz`. Se verificó contra
`S3Private.UploadObjectAsync` del front: con `gzip=false` se manda
`ContentBody=textContent` tal cual, así que guardar gzip binario real rompería la
lectura que ya hace Angular hoy.

---

## `CargaReporteAnalytics` — la solicitud de carga (registro intermedio)

Colección `CargasReportesAnalytics`. Vive en estado `PENDIENTE` hasta que se confirma o
expira; nunca es el documento final, es el puente entre "pedí una URL" y "confirmé la
publicación".

| Campo | Tipo | Notas |
| ----- | ---- | ----- |
| `CargaId` | `string` | También es el `_id` — único nativo, sin índice extra |
| `ReporteId` | `string?` | Null si es un reporte nuevo |
| `IdentidadOrigen` | `string` | El `sub` del JWT — usado para las comprobaciones de dueño |
| `Autor` | `string` | Email/username, para auditoría legible sin cruzar `Users` |
| `Estado` | `string` | `PENDIENTE` \| `COMPLETADA` \| `RECHAZADA` |
| `NombreArchivoDeclarado`, `TamanoDeclaradoBytes` | — | Lo que mandó el cliente en `solicitar-carga`, no verificado todavía |
| `FechaSolicitud` | `DateTimeOffset` | Base para calcular expiración (10 min) |
| `EnProcesamiento` | `bool` | Candado de exclusión mutua — ver [`ConfirmarCargaUseCase`](../casos-uso/confirmar-carga-usecase.md) |

Campos adicionales que solo escribe `CargaRepository` directamente en Mongo (no forman
parte del record de dominio, son auditoría pura): `FechaInicioProcesamiento`,
`FechaFinalizacion`, `Hallazgos`, `ReporteIdPublicado`, `VersionIdPublicado`,
`ErrorProcesamiento`.

---

## `IdentidadCaller`

```csharp
public sealed record IdentidadCaller(string Sub, string Autor);
```

La identidad ya resuelta y autorizada, producto de `AutorizacionUsuario.ExigirRolAsync`.
`Sub` viene del JWT (sin verificar firma); `Autor` viene del documento de `Users` (email
si existe, si no username) — **nunca del body del cliente**. Es el único vehículo por el
que la identidad de quien publica llega a los casos de uso.

---

## Enums

### `EstadoCarga`
```csharp
public enum EstadoCarga { Pendiente, Completada, Rechazada }
```
Su representación persistida en Mongo es siempre el string en mayúsculas
(`"PENDIENTE"`, etc.) — el enum de C# es solo para el código, `CargaRepository` escribe
y lee el string directamente.

### `EstadoReporte`
```csharp
public enum EstadoReporte { Activo, Borrador, Archivado }
```
Persistido como `ACTIVO` \| `BORRADOR` \| `ARCHIVADO`.

---

## `HallazgoValidacion`

```csharp
public sealed record HallazgoValidacion(string Regla, string Motivo, int Linea = 0, string Extracto = "");
```

Un motivo de rechazo de contenido. `Linea` usa base 1 (cero si el hallazgo no corresponde
a una línea concreta, por ejemplo `TAMANO_EXCEDIDO`). `Extracto` es el fragmento del HTML
que disparó la regla, recortado a 120 caracteres — se devuelve al dueño del reporte en la
respuesta 422, **nunca se manda a logs** (podría contener contenido sensible del
reporte).

---

## `ReglaProhibida`

```csharp
public sealed record ReglaProhibida(string Regla, Regex Patron, string Motivo);
```

Empareja un código de regla con su regex compilada y el motivo legible — así están
declaradas las 8 reglas de seguridad en `ReporteValidadorService`
(ver [infraestructura/servicios.md](../infraestructura/servicios.md#reportevalidadorservice)).
