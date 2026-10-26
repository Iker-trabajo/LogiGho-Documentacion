## Autor: Iker Acevedo

Fecha creacion: 2026-09-21

Estado: desarrollo

## Lambda: ApiLambdaPublicarReporteAnalytics

**Accionador:** API Gateway HTTP (v2) — `g3iz6qk3f0`, 2 rutas
**AOT:** No (runtime `dotnet10` managed)

---

## ¿Qué hace?

Mueve al backend el flujo de "publicar/versionar reportes analíticos" del dominio
**Datos**, que hasta ahora corría 100% en el cliente (Angular subía el HTML directo a
S3 y escribía Mongo directo, sin ningún control server-side). Con esto, tanto Angular
como cualquier integración externa del área de datos quedan validados por el mismo
código, con las mismas reglas de seguridad — antes cada cliente podía aplicarlas
distinto o saltárselas por completo.

El HTML nunca pasa por Lambda ni por API Gateway: se usa una **URL prefirmada de S3 en
dos pasos** (`solicitar-carga` → `PUT` directo a S3 → `confirmar-carga`), así se evita
el límite de payload de API Gateway (6 MB) para reportes de hasta 15 MB.

---

## Por qué evento API Gateway V2, y no V1 como el resto del repo

Toda otra lambda del repo detrás de API Gateway usa el tipo de evento V1
(`APIGatewayProxyRequest`/`Response`), porque casi todo el repo cuelga del API Gateway
REST clásico. Esta es la única excepción: va sobre el Gateway **HTTP API v2**
(`g3iz6qk3f0`) con `PayloadFormatVersion: "2.0"` explícito en el CFN, así que el handler
usa `APIGatewayHttpApiV2ProxyRequest`/`Response` y lee la ruta/método desde
`RequestContext.Http.Method`/`RawPath` en vez de los campos planos de V1.

Se detectó en la primera prueba real: con el tipo V1, `request.HttpMethod` quedaba
siempre vacío contra un evento V2 real, y toda petición caía al `405 RUTA_NO_ADMITIDA`
por defecto sin importar el método enviado. HTTP API v2 también puede emular el formato
V1 (`PayloadFormatVersion: "1.0"`, como se hizo manualmente para `ApiLambdaGetObjectMetadataAOT`
al resolver la visualización — ver [Infraestructura y despliegue](#infraestructura-aws-y-despliegue)),
pero acá se decidió usar el formato nativo del Gateway en vez de forzar compatibilidad
con el estándar REST viejo.

---

## Los dos endpoints

| Método | Ruta | Caso de uso | Auth |
| ------ | ---- | ----------- | ---- |
| `POST` | `/reportes-analytics/solicitar-carga` | [`SolicitarCargaUseCase`](casos-uso/solicitar-carga-usecase.md) | header `Token` (sub + rol en `Users`) |
| `POST` | `/reportes-analytics/confirmar-carga/{cargaId}` | [`ConfirmarCargaUseCase`](casos-uso/confirmar-carga-usecase.md) | header `Token` (sub + rol en `Users`) |

Entre uno y otro, el cliente hace un tercer paso que **no pasa por esta Lambda**: un
`PUT` directo a la URL prefirmada que devuelve el primer endpoint.

---

## Flujo completo, punta a punta

```
Cliente                         Lambda                          Mongo / S3
  |--POST solicitar-carga------->|                                  |
  |                              |--autoriza (Users, por sub)------->|
  |                              |--valida declaracion (nombre/tamano)
  |                              |--si trae ReporteId, valida que exista
  |                              |--inserta Carga PENDIENTE--------->| CargasReportesAnalytics
  |                              |--firma URL PUT (expira 10 min)
  |<--CargaId + UrlSubida + ExpiraEn--|                              |
  |                                                                   |
  |--PUT HTML crudo a UrlSubida (directo a S3, sin pasar por Lambda)->| staging/reportes-analytics/{cargaId}/original.html
  |                                                                   |
  |--POST confirmar-carga/{cargaId}->|                                |
  |                              |--autoriza + valida dueno/estado/expiracion
  |                              |--adquiere la carga (compare-and-set)->|
  |                              |--descarga staging (limite real de bytes)
  |                              |--valida tamano real + UTF-8 + las 8 reglas de contenido
  |                              |--gzip + SHA-256 + verifica roundtrip local
  |                              |--sube version definitiva (If-None-Match:*)->| reportes-analytics/{reporteId}/{versionId}.html.gz
  |                              |--relee la version subida y reverifica el hash
  |                              |--$push de la version + $set metadatos--->| ReportesAnalytics
  |                              |--marca Carga COMPLETADA----------->| CargasReportesAnalytics
  |                              |--borra el staging (finally, siempre)-->|
  |<--200 { Valido:true, ReporteId, VersionId }--|                    |
```

Ver el detalle de cada paso en [`SolicitarCargaUseCase`](casos-uso/solicitar-carga-usecase.md)
y [`ConfirmarCargaUseCase`](casos-uso/confirmar-carga-usecase.md).

### Decide nuevo reporte vs. nueva versión de uno existente

Lo decide quien llama, mandando o no `ReporteId` en `solicitar-carga` — la API no
adivina por nombre de archivo ni por ningún otro heurístico (sería frágil). Con
`ReporteId`: valida que exista (`404 REPORTE_NO_EXISTE` si no), y al confirmar hace
`$push` de una versión nueva + mueve `VersionActiva`, sin tocar el historial anterior.
Sin `ReporteId`: crea un reporte nuevo con `Guid.NewGuid()`.

---

## Request / Response

### 1. `POST /reportes-analytics/solicitar-carga`

```json
{ "NombreArchivo": "ventas.html", "TamanoDeclaradoBytes": 81883, "ReporteId": null }
```

| Campo | Tipo | Requerido | Descripción |
| ----- | ---- | --------- | ----------- |
| `NombreArchivo` | `string` | Sí | Debe terminar en `.html`/`.htm` |
| `TamanoDeclaradoBytes` | `long` | Sí | Solo para rechazo temprano — el tamaño real se valida leyendo S3, nunca se confía en este valor |
| `ReporteId` | `string?` | No | Si viene, versiona ese reporte existente; si no, crea uno nuevo |

**200 OK:**
```json
{ "CargaId": "cee3c8aa-...", "UrlSubida": "https://logigho-plantillas-preprod.s3...", "ExpiraEn": "2026-09-21T16:33:08Z" }
```

**422** (declaración inválida — extensión, tamaño):
```json
{ "Valido": false, "Hallazgos": [{ "Regla": "EXTENSION_INVALIDA", "Motivo": "...", "Linea": 0, "Extracto": "..." }] }
```

### 2. `PUT` a `UrlSubida`

Directo a S3, sin pasar por esta Lambda. Body: el HTML **crudo** (UTF-8, sin gzip, sin
base64). El método debe ser exactamente `PUT` — la firma SigV4 incluye el verbo; abrir
el link en el navegador (`GET`) da `SignatureDoesNotMatch`, es esperado. Sin headers
extra: cualquier header no incluido al firmar también invalida la firma.

### 3. `POST /reportes-analytics/confirmar-carga/{cargaId}`

```json
{
  "Nombre": "Ventas", "Descripcion": "", "Categoria": "Ventas",
  "Estado": "ACTIVO", "RolesPermitidos": ["Jefe Datos", "CEO"], "Notas": ""
}
```

| Campo | Tipo | Requerido | Descripción |
| ----- | ---- | --------- | ----------- |
| `Nombre` | `string` | Sí | Máximo 200 caracteres |
| `Descripcion` | `string` | Sí (puede ser vacía) | Máximo 4000 caracteres |
| `Categoria` | `string` | Sí | Máximo 100 caracteres |
| `Estado` | `string` | Sí | `ACTIVO` \| `BORRADOR` \| `ARCHIVADO` |
| `RolesPermitidos` | `string[]` | Sí | Entre 1 y 100 roles, máximo 100 caracteres cada uno |
| `Notas` | `string` | Sí (puede ser vacía) | Máximo 4000 caracteres |

**200 OK** (publicado):
```json
{ "Valido": true, "ReporteId": "8c7d4215-...", "VersionId": "93e9b2e6-..." }
```

**422** (contenido rechazado por el validador):
```json
{ "Valido": false, "Hallazgos": [ ... ] }
```

### Errores (`{"Error":"CODIGO"}`)

| Status | Código | Cuándo |
| ------ | ------ | ------ |
| 400 | `BODY_INVALIDO` / `METADATOS_INVALIDOS` / `REPORTE_ID_INVALIDO` / `CARGA_ID_INVALIDO` | Contrato mal formado |
| 401 | `IDENTIDAD_AUSENTE` | Sin header `Token`, o `sub` no decodificable |
| 401 | `SIN_ROL_PERMITIDO` | El usuario no existe en `Users`, o no tiene ninguno de los 3 roles |
| 403 | `CARGA_AJENA` | El `sub` que confirma no es el dueño de esa carga |
| 404 | `REPORTE_NO_EXISTE` / `CARGA_NO_EXISTE` | IDs sintácticamente válidos pero inexistentes |
| 409 | `CARGA_NO_PENDIENTE` / `CARGA_CAMBIO_DURANTE_CONFIRMACION` / `REPORTE_CAMBIO_DURANTE_CARGA` / `OBJETO_STAGING_NO_EXISTE` | Carreras concurrentes o estado inconsistente |
| 410 | `CARGA_EXPIRADA` | Pasaron los 10 minutos desde `solicitar-carga` |
| 500 | `INTEGRIDAD_INVALIDA` / `ERROR_INTERNO` | Hash no coincide en algún punto del roundtrip, o excepción no prevista |
| 503 | `LIMPIEZA_PENDIENTE` | Falló el borrado del objeto de staging tras confirmar (la publicación ya quedó completa; solo falló la limpieza) |

Respuestas siempre en clases tipadas (nunca objetos anónimos), PascalCase,
`Cache-Control: no-store`, sin headers CORS (servidor a servidor).

---

## Seguridad y autorización

No hay Cognito Authorizer, no hay validación de firma JWT, no hay colección de
integraciones externas ni kill-switch propio — **decisión explícita**, no lo que
proponía el diseño original (ver [ADR-001](adr/ADR-001-autenticacion-sub-sin-firma-mas-roles-en-users.md)).

1. El handler decodifica el `sub` del JWT del header `Token` (`DecodificadorJwt`) —
   **sin verificar firma, issuer, audience ni expiración**.
2. Busca ese `sub` en `Users.cognitoId` (`UsuarioRepository`).
3. Exige que el documento tenga alguno de los roles `Jefe Datos`, `Desarrollador` o
   `CEO` (`AutorizacionUsuario.ExigirRolAsync`).
4. Sin usuario o sin rol permitido → `401 SIN_ROL_PERMITIDO`, mismo código en ambos
   casos — no se revela cuál de los dos pasó.

Bloquear a alguien = quitarle el rol en `Users`, igual que en el resto de la
plataforma. No hay nada especial que administrar para este lambda en particular.

`Autor` (auditoría en `CargasReportesAnalytics`, y `CreadoPor`/`ActualizadoPor`/`Autor`
de `ReportesAnalytics`) siempre sale de `Users` (email o username), nunca del body que
manda el cliente — ver [dominio/modelos.md](dominio/modelos.md#identidadcaller).

---

## Validación de contenido (las 8 reglas)

Port literal de las regex del sanitizador Angular (`endurecerHtml`), verificadas 8/8
idénticas contra el archivo fuente. Corren en dos pasadas — por línea (con número de
línea real para el hallazgo) y sobre el documento completo (para atacar tokens o
atributos partidos entre saltos de línea, que el escaneo por línea no vería):

| Regla | Qué bloquea |
| ----- | ----------- |
| `RECURSO_EXTERNO` | `src`/`href`/`data`/`action`/`poster` apuntando a `http(s)://` o `//` |
| `IMPORT_CSS_EXTERNO` | `@import` de una hoja de estilos externa |
| `SALIDA_DE_RED` | `fetch`, `XMLHttpRequest`, `WebSocket`, `EventSource`, `sendBeacon` |
| `IMPORT_DINAMICO` | `import(...)` dinámico |
| `ETIQUETA_PROHIBIDA` | `<form>`, `<iframe>`, `<object>`, `<embed>`, `<frame>`, `<frameset>`, `<portal>` |
| `ACCESO_AL_ANFITRION` | `document.cookie`, `window.parent/top/opener`, `postMessage`, `localStorage`, `sessionStorage`, `indexedDB` |
| `NAVEGACION` | `window.open`, `location.href/replace/assign`, `document.location =` |
| `META_REFRESH` | `<meta http-equiv="refresh">` |

Más validaciones estructurales: extensión `.html`/`.htm`, debe empezar con
`<!doctype html>` o `<html>`, tamaño real ≤ 15 MB (nunca el declarado por el cliente),
UTF-8 estricto, presupuesto de 10s de CPU para el escaneo completo
(`VALIDACION_AGOTADA` si se agota — nunca corre sin límite), máximo 1000 hallazgos
reportados (`LIMITE_HALLAZGOS`).

**Límite reconocido, no resuelto acá:** el validador escanea con regex, no ejecuta el
HTML. JavaScript inline/`eval` sigue permitido — se prueba explícitamente el caso
`window['fe'+'tch']` del spec de Angular, y ese *pasa* el validador a propósito (no se
puede demostrar ausencia de exfiltración ante código ofuscado con regex). El consumidor
final debe seguir renderizando con sandbox y CSP restrictiva.

---

## Variables de entorno

| Variable | Contenido |
| -------- | --------- |
| `MONGODB_CONNECTION_STRING` | Connection string **cifrado AES256-ECB** (`Seguridad.Encripcion.EncripcionAES`, mismo estándar del resto del repo) — el lambda lo desencripta en el arranque, antes de conectar. Nunca en texto plano en el secret de GitHub. |
| `DATABASE_NAME` | `LogighoDB` — misma base donde vive `Users`. |
| `REPORTES_S3_BUCKET` | `logigho-plantillas-preprod` en PreProd. |

AWS SDK usa la cadena estándar de credenciales del rol de ejecución de Lambda, sin
access keys en código. Mongo: TLS obligatorio con validación de certificado (el arranque
lanza si `UseTls` es falso o `AllowInsecureTls` es verdadero), `ReadPreference.Primary`,
`WriteConcern.WMajority`, `RetryWrites=false`, timeouts de conexión/selección de 5s.

---

## Infraestructura AWS y despliegue

CloudFormation puro (`.preprod-hub/infra/templates/datos-infra-preprod.yaml`), dominio
**Datos**:

- Rol IAM propio (`ApiLambdaPublicarReporteAnalyticsRole`), mínimo privilegio: solo
  CloudWatch Logs y `s3:PutObject/GetObject/DeleteObject` acotado a los prefijos
  `staging/reportes-analytics/*` y `reportes-analytics/*`.
- Integración `AWS_PROXY` con `PayloadFormatVersion: "2.0"` — la única integración V2
  del repo (ver [más arriba](#por-qué-evento-api-gateway-v2-y-no-v1-como-el-resto-del-repo)).
- Dos rutas, `AuthorizationType: NONE` — igual que todas las demás rutas de este
  Gateway (confirmado con `aws apigatewayv2 get-routes`, no hay Authorizer en ningún
  lado de la plataforma).

Pipeline: `.github/workflows/deploy-datos-preprod.yml`, dos jobs encadenados (`test` →
`deploy`, con `test` como gate — si falla, no llega a desplegar), OIDC sin llaves
estáticas, notificaciones a Slack en cada etapa (mismo estándar del resto de dominios).

**`GitHubActionsDeployRole`** (`cuenta-infra-preprod.yaml`, transversal, no exclusivo de
Datos) necesitó 4 Sid nuevos la primera vez que se desplegó este dominio, porque fue la
primera vez que el pipeline crea un rol IAM completo por CFN (los demás dominios reusan
roles pre-creados a mano): manejo del rol de ejecución propio, permiso de invocación
desde API Gateway, y manejo de integraciones/rutas sobre `g3iz6qk3f0`.

**`ApiLambdaGetObjectMetadataAOT`** — lambda hermana necesaria para que Angular pueda
*leer* el HTML ya publicado (esta lambda de acá solo escribe). No existía en la cuenta
PreProd; se replicó a mano desde Producción (mismo paquete `dotnet8`, sin el `VpcConfig`
de prod porque no aplica acá) con ruta `POST /getObject` y `PayloadFormatVersion: "1.0"`
(formato V1, porque su código espera el evento REST clásico como el resto del repo).
Pendiente formalizar en IaC — hoy es un recurso manual, sin protección de drift.

---

## Testing

`ApiLambdaPublicarReporteAnalytics.Tests` — xUnit + Moq, sin red real (Mongo/S3
mockeados). 96 tests: paridad de las 8 reglas contra el archivo fuente de Angular,
reglas individuales, tamaño/UTF-8 real, identidad y roles, toma atómica de la carga
(compare-and-set), versionado/schema Mongo, gzip/base64/hash, corrupción, limpieza de
staging, y los tipos de evento V2 del handler.

```powershell
dotnet build .\ApiLambdaPublicarReporteAnalytics\ApiLambdaPublicarReporteAnalytics.csproj
dotnet test .\ApiLambdaPublicarReporteAnalytics.Tests\ApiLambdaPublicarReporteAnalytics.Tests.csproj
```

---

## Historial de cambios

| Fecha | Autor | Cambio |
| ----- | ----- | ------ |
| 2026-09-19 | Iker Acevedo | Diseño y construcción inicial: Clean Architecture completa, presigned URL en 2 pasos, port de las 8 reglas de validación, compare-and-set en carga y en versionado. |
| 2026-09-20 | Iker Acevedo | Rediseño completo del modelo de autorización: de un diseño con Cognito Authorizer + colección `IntegracionesExternas` a decodificar el `sub` sin verificar firma y exigir rol en `Users` — ver [ADR-001](adr/ADR-001-autenticacion-sub-sin-firma-mas-roles-en-users.md). Construcción del pipeline completo (CloudFormation + GitHub Actions) para el dominio Datos. |
| 2026-09-21 | Iker Acevedo | Fix crítico: el handler usaba tipos de evento API Gateway V1 contra una integración V2 real, causando `405` en toda petición — migrado a `APIGatewayHttpApiV2ProxyRequest`/`Response`. `MONGODB_CONNECTION_STRING` migrado a cifrado AES256-ECB (estándar del repo). Respuestas de error migradas de objetos anónimos a records tipados (`ErrorResponse`, `ValidacionRechazadaResponse`). Nuevo campo `Autor` en `CargasReportesAnalytics` para auditoría legible. Despliegue manual de `ApiLambdaGetObjectMetadataAOT` en PreProd para desbloquear la visualización desde Angular. |

---

## Observaciones

- Sin transacción distribuida S3/Mongo: un fallo ambiguo entre subir a S3 y escribir
  Mongo deja la carga `PENDIENTE`/`EnProcesamiento=true` con
  `ErrorProcesamiento=REVISION_REQUERIDA` — no se reabre ni reintenta sola, requiere
  conciliación manual.
- Una URL prefirmada ya emitida no se revoca al bloquear a un usuario (quitarle el
  rol): puede seguir recreando el objeto de staging hasta que expire (10 min). El
  bloqueo impide la siguiente `solicitar-carga` y cualquier `confirmar-carga`, pero no
  cancela un `PUT` ya en curso.
- TODO no bloqueante: regla de ciclo de vida en S3 sobre el prefijo `staging/` (limpieza
  de huérfanos por fallos ambiguos), índice único en `ReporteId` dentro de
  `ReportesAnalytics`.
