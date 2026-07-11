## Autor: Iker Acevedo
Fecha creacion: 2026-07-09

Estado: produccion

## Lambda: ApiLambdaCrudGenericoAOT


**Accionador:** API Gateway

**AOT:** Sí 

---

## ¿Qué hace?

Expone un CRUD genérico sobre cualquier colección de MongoDB. El nombre de la colección llega por query param (`coleccion`) y la operación se decide según el método HTTP (`GET` / `POST` / `PUT` / `PATCH` / `DELETE`). Es la lambda más consumida de la plataforma, por eso está compilada en **Native AOT** para minimizar el *cold start*.

Los datos de lectura viajan **comprimidos** (gzip o zstd) y en Base64 dentro del campo `Resultado`. En **producción el acceso está protegido por API Gateway**, que valida el **JWT de Cognito** mediante un *authorizer* antes de invocar la lambda. La lambda **no re-valida la firma del token** (confía en la validación del Gateway); `ValidadorPipeline` es una defensa en profundidad adicional sobre el `pipeline`.

**Novedades de esta versión** (aditivas, sin romper el comportamiento previo):
1. **`campos`** — proyección de MongoDB: devolver solo los campos pedidos.
2. **`pipeline`** — ejecutar un `$aggregate` (dashboards), con `MaxTime` y sin `AllowDiskUse`.
3. **`ValidadorPipeline`** — seguridad: rechaza operadores peligrosos en el pipeline.

---

## Accionador

| Método | Ruta | Operación | Autenticación |
| ------ | ---- | --------- | ------------- |
| `GET` | API Gateway — `ANY /generico` | Consultar (Find) o agregar (`pipeline`) | JWT Cognito (API Gateway) |
| `POST` | API Gateway — `ANY /generico` | Insertar documentos | JWT Cognito (API Gateway) |
| `PUT` | API Gateway — `ANY /generico` | Actualizar (por `_id` o masivo con filtro) | JWT Cognito (API Gateway) |
| `PATCH` | API Gateway — `ANY /generico` | Consulta anidada (filtros OR + regex) | JWT Cognito (API Gateway) |
| `DELETE` | API Gateway — `ANY /generico` | Eliminar filtrados o **la colección entera** | JWT Cognito (API Gateway) |

> ⚠️ La integración de API Gateway usa **Payload format 1.0** (`APIGatewayProxyRequest` v1); con 2.0, `HttpMethod` llega vacío. En **producción**, un **authorizer de Cognito** valida el JWT antes de invocar la lambda — sin token válido, API Gateway responde `401`/`403` y la lambda no se ejecuta.

---

## Request

### Headers

| Header | Requerido | Descripción |
| ------ | --------- | ----------- |
| `Authorization` | Sí (producción) | `Bearer <IdToken>` de Cognito. El *authorizer* del API Gateway lo valida antes de invocar la lambda |
| `Token` | Alternativa | IdToken de Cognito (según el *authorizer* configurado) |
| `Content-Type` | POST / PUT | `application/json` |

> El token de Cognito se obtiene desde `APILambdaObtenerToken`. La lambda genérica **no lee ni valida** el token — es el API Gateway quien lo exige y valida.

### Query Parameters — Reservados (no se usan como filtro)

| Parámetro | Aplica a | Tipo | Default | Descripción |
| --------- | -------- | ---- | ------- | ----------- |
| `coleccion` | Todos | `string` | — (obligatorio) | Colección MongoDB sobre la que se opera |
| `page` | GET, PATCH | `int` | `1` | Página a consultar |
| `mcomp` | GET, PATCH | `int` | `1` | Compresión: `1`=gzip, `2`=zstd |
| `lote` | GET | `int` | `4000` (mcomp=1) / `8000` (mcomp≠1) | Tamaño de página (registros por página) |
| `fechasFiltro` | GET | `string` | — | Rango de fechas por `_id`. **Formato exacto: `YYYYMMDD-YYYYMMDD`** (ej: `20250101-20251106`) |
| `campos` | GET | `string` CSV | — | **(NUEVO)** Proyección: solo esos campos (+`_id`) |
| `pipeline` | GET | `string` Base64 | — | **(NUEVO)** Ejecuta `$aggregate` en vez de `Find` |
| `blnUpdateAll` | PUT | `bool` | `false` | Si `true`, actualiza todos los que cumplan el filtro |
| `_id` | PUT | `string` | — | ObjectId del documento a actualizar (si no hay `blnUpdateAll`) |

### Filtros dinámicos — **cualquier campo de la colección**

Todo query param que **no** sea reservado se convierte en un filtro sobre ese campo. Puedes pasar **cualquier campo** que exista en los documentos. El **valor** se interpreta así (GET — `ObtenerDocumentoParaleloAsync`):

| Forma del valor | Ejemplo | Cómo filtra en Mongo |
| --------------- | ------- | -------------------- |
| Con comas | `Estado=Entregada,Devuelta` | `$in` (cualquiera de los valores) |
| Numérico entero | `IdCarga=10086` | `Eq(campo, 10086)` **OR** `Regex(campo, /10086/i)` — encuentra el campo sea número o texto |
| Numérico decimal | `Valor=1.5` | `Eq(campo, 1.5)` |
| Texto | `Tienda=Norte` | `Eq(campo, "Norte")` — **coincidencia exacta** |

- Los filtros se combinan con **AND** (todos deben cumplirse).
- En un valor con comas, si mezcla números y texto, arma un `$in` combinado (números como `long` y como string, y textos) unidos con OR — tolera colecciones donde el mismo campo es a veces número y a veces string.
- Los filtros `Tienda`, `NombreTienda`, `Promotor` con valor que contenga `"todas"` se **ignoran** (los quita `Function.cs` antes de llamar al repo).

### Búsqueda global (`*` o `dync`)

| Parámetro | Ejemplo | Qué hace |
| --------- | ------- | -------- |
| `*` o `dync` | `dync=3001234567` | Busca ese valor en **todos los campos** del primer documento de la colección (hasta 42 campos), respetando el tipo (string/int/double). Combina con **OR** |


### Ordenamiento

Siempre **descendente por `_id`** (los documentos más recientes primero). No es configurable desde el request en esta lambda.

### Ejemplos de request

```
# Filtro exacto + proyección
GET /generico?coleccion=PedidosInter&Estado=Entregada&campos=Estado,Tienda&lote=50&mcomp=1

# Varios valores (IN) + rango de fechas
GET /generico?coleccion=PedidosInter&Estado=Entregada,Devuelta&fechasFiltro=20260101-20260630

# Búsqueda global
GET /generico?coleccion=PedidosInter&dync=INTERRAPIDISIMO

# Aggregation (pipeline en Base64)
GET /generico?coleccion=PedidosInter&pipeline=W3siJGdyb3VwIjp7Il9pZCI6IiRFc3RhZG8ifX1d&mcomp=2

# Insertar
POST /generico?coleccion=PedidosInter
Body: [ { "Campo": "valor" } ]

# Actualizar uno por _id
PUT /generico?coleccion=PedidosInter&_id=66a1...
Body: { "Estado": "Entregada" }

# Actualizar masivo con filtro tipado
PUT /generico?coleccion=PedidosInter&blnUpdateAll=true&Estado=Cargado
Body: { "Estado": "Anulada" }

# Eliminar filtrados  (¡sin filtros elimina la COLECCIÓN entera!)
DELETE /generico?coleccion=PedidosInter&Estado=Prueba
```


---

## Response

Todas las respuestas usan el contrato genérico **`RespuestaGeneral<T>`**:

| Campo | Tipo | Descripción |
| ----- | ---- | ----------- |
| `Error` | `bool` | `false` en éxito, `true` en error (default `false`) |
| `Mensaje` | `string?` | Texto descriptivo. `null` en lecturas OK; con texto en escrituras y errores |
| `NumeroPaginas` | `int` | Total de páginas según `lote`. En aggregation = `1`. En POST/PUT = `0` |
| `TotalRegistros` | `int` | Documentos que cumplen el filtro (o que devolvió el pipeline). En POST/PUT = `0` |
| `Resultado` | `T?` | Varía según el método (ver tabla) |

El tipo de `Resultado` cambia según la operación:

| Método | Tipo | Contenido de `Resultado` |
| ------ | ---- | ------------------------ |
| GET / PATCH | `string` | Array de documentos JSON, **comprimido** (gzip/zstd) y en Base64 |
| POST | `List<object>` | `null` (la confirmación va en `Mensaje`) |
| PUT | `List<object>` | `null` (la confirmación va en `Mensaje`) |
| DELETE | `long` | Nº de documentos eliminados |

### Ejemplos por método

**GET / PATCH (200)** — `Resultado` comprimido:
```json
{ "Error": false, "Mensaje": null, "NumeroPaginas": 27, "TotalRegistros": 208088, "Resultado": "KLUv/WClAs0L..." }
```
Al descomprimir `Resultado` obtienes un **array de documentos**. Como la lambda es **genérica, los campos dependen de la colección**. Ejemplo para `PedidosInter`:
```json
[
  {
    "_id": "66a1f2...", "IdCarga": "10086", "Estado": "Entregada", "Tienda": "...",
    "Transportadora": "INTERRAPIDISIMO", "FormaDePago": "CONTADO", "Direccion": "CRA 31...",
    "Alto": "10.0", "Ancho": "10.0", "Largo": "10.0", "Peso": "1.0", "AplicaContraPago": false,
    "TipoDeEntrega": 1, "Asesor": "Alejandra", "DiceContener": "ROPA", "Seguro": "SI",
    "ValorDeclarado": 124900, "FechaCreacion": "..."
  }
]
```

**Aggregation (`pipeline`) (200):**
```json
{ "Error": false, "Mensaje": null, "NumeroPaginas": 1, "TotalRegistros": 25, "Resultado": "KLUv/..." }
```
`Resultado` descomprimido = salida del pipeline, ej: `[ { "_id": "Entregada", "total": 8123 }, ... ]`

**POST (200):**
```json
{ "Error": false, "Mensaje": "Documento Insertado con Exito.", "NumeroPaginas": 0, "TotalRegistros": 0, "Resultado": null }
```

**PUT por `_id` (200):**
```json
{ "Error": false, "Mensaje": "Documento actualizado con éxito.", "NumeroPaginas": 0, "TotalRegistros": 0, "Resultado": null }
```
Con `blnUpdateAll=true`: `"Documentos actualizados con exito segun filtro."`

**DELETE con filtros (200):**
```json
{ "Error": false, "Mensaje": "Se eliminaron 12 documento(s) con los filtros especificados.", "NumeroPaginas": 0, "TotalRegistros": 0, "Resultado": 12 }
```
Sin filtros: `"Colección eliminada completamente."` (⚠️ elimina la colección entera)

### Errores

| Código | Cuándo | `Mensaje` ejemplo |
| ------ | ------ | ----------------- |
| `400` | **(NUEVO)** `pipeline` no es Base64/JSON válido | "El parámetro 'pipeline' no es un Base64/JSON válido." |
| `400` | **(NUEVO)** `pipeline` con operadores prohibidos | "El pipeline contiene etapas u operadores no permitidos." |
| `405` | Método HTTP no soportado | "Método HTTP no soportado." |
| `500` | PUT sin `_id` ni `blnUpdateAll` | "Debe enviar el Id del Registro o activar blnUpdateAll." |
| `500` | Excepción no controlada | "Excepcion no Controlada: {detalle}" |

```json
{ "Error": true, "Mensaje": "El pipeline contiene etapas u operadores no permitidos." }
```

---

## NUEVO — Proyección de campos (`campos`)

`Function.cs` parte el CSV y lo pasa como `camposProyectados`. En `DocumentRepository.ObtenerDocumentoParaleloAsync` se arma un `Projection.Include("_id")` + cada campo. Sin `campos`, se devuelve el documento completo (comportamiento histórico).

---

## NUEVO — Aggregation Pipeline (`pipeline`)

- Viaja en **Base64** (url-safe o estándar; se normaliza con `Utils.DecodificarBase64UrlSafe`).
- Se parsea a `List<BsonDocument>`, se valida con `ValidadorPipeline`, y se ejecuta con `EjecutarAggregateAsync`.
- **Protecciones de ejecución:** `AllowDiskUse = false` y `MaxTime = 20000 ms` (corta pipelines lentos antes del timeout de 30 s de la Lambda).

```json
[
  { "$match": { "FechaCreacion": { "$gte": "2026-01-01" } } },
  { "$group": { "_id": "$Estado", "total": { "$sum": 1 }, "ventas": { "$sum": "$ValorFlete" } } },
  { "$sort": { "total": -1 } }
]
```

---

## NUEVO — Seguridad del pipeline (`ValidadorPipeline`)

Recorre **recursivamente** todo el árbol BSON y rechaza (con `400`) operadores peligrosos a **cualquier profundidad**. También rechaza etapas vacías o pipeline nulo.

| Categoría | Operadores |
| --------- | ---------- |
| Escritura / persistencia | `$out`, `$merge` |
| Lectura de otras colecciones | `$lookup`, `$graphLookup`, `$unionWith` |
| Ejecución de JavaScript en la BD | `$function`, `$accumulator`, `$where` |
| Fuga de metadatos | `$currentOp`, `$listLocalSessions`, `$planCacheStats`, `$collStats`, `$indexStats` |

---

## Filtro por fechas (`fechasFiltro`) — detalle

- **Formato:** `YYYYMMDD-YYYYMMDD` (ej: `20250101-20251106`). Cualquier otro formato se **ignora silenciosamente** (no falla).
- Convierte cada fecha a un `ObjectId` sintético (el timestamp va en los primeros 4 bytes del `_id`) y filtra `_id >= inicio AND _id < finDelDíaSiguiente`.
- **Aprovecha el índice `_id`** — no requiere índice de fecha adicional.
- Excepción: en la colección `UnionPedidosLiquidaciones` filtra por `PedidoId` en vez de `_id`.

---

## Métodos de compresión

El código (`Utils.cs`) implementa **3 algoritmos**; el flujo del handler usa **2** (según `mcomp`):

| `mcomp` | Algoritmo | Método usado | Firma Base64 | Descompresión cliente |
| ------- | --------- | ------------ | ------------ | --------------------- |
| `1` (default) | **gzip** | `CompressObjectAsync` (`GZipStream`, nivel `Optimal`) | `H4sI...` (`1f 8b`) | `pako.ungzip` |
| `2` | **zstd** | `CompressObjectZstdAsync` (`ZstdSharp.Zstd.Compress`) | `KLUv/...` (`28 b5`) | `fzstd.decompress` |
| — | **brotli** | `CompressWithBrotli` (`BrotliStream`) — **disponible pero NO usado** en el handler | — | — |

- El JSON se serializa con **Newtonsoft** (`JsonSerializer`) directo al stream de compresión.
- Métodos inversos disponibles: `DecompressObjectAsync` (gzip), `DecompressObjectZstdAsync` (zstd), `DecompressWithBrotli`.
- Utilidades extra (no usadas por el flujo principal): `CompressObjetoAsync`, `CompressStringAsync`, `DecompressObjetoUnicoAsync`.

---

## Contrato del repositorio (`IDocumentRepository`)

| Método | Usado por | Qué hace |
| ------ | --------- | -------- |
| `ObtenerDocumentoParaleloAsync` | GET | Find con filtros (AND), fechas, proyección (`campos`), paginación; sort `_id` desc. Cuenta total en paralelo |
| `ObtenerDocumentoParaleloAnidadoAsync` | PATCH | Igual pero filtros con **OR** y strings por **regex** (contiene); soporta filtros anidados |
| `EjecutarAggregateAsync` | GET (`pipeline`) | `$aggregate` con `MaxTime` 20 s y `AllowDiskUse=false` |
| `InsertarDocumentosAsync` | POST | `InsertMany` |
| `ActualizarDocumentoAsync` | PUT (`_id`) | `UpdateOne` con `$set`, `IsUpsert=false` |
| `ActualizarCampoAllAsync` | PUT (`blnUpdateAll`) | `UpdateMany` con `$set`; filtros tipados `campo:tipo=valor` |
| `EliminarDocumentosFiltradosAsync` | DELETE (con filtro) | `DeleteMany`; devuelve nº eliminados |
| `EliminarColeccionAsync` | DELETE (sin filtro) | **`DropCollection`** — elimina la colección completa |

---

## Variables de entorno

| Variable | Descripción | Valor ejemplo |
| -------- | ----------- | ------------- |
| `CADENA_CONEXION` | Cadena MongoDB **encriptada AES-256-ECB / PKCS7 / HEX**. Se descifra en `DocumentRepository.Create()` con `EncripcionAES.DecryptAES256ECB` | String hex |
| `DATABASE_NAME` | Base de datos MongoDB | `"LogighoDB"` |


---

## Configuración Lambda

| Parámetro | Valor |
| --------- | ----- |
| Runtime | `dotnet8` (self-contained AOT) |
| Handler | `ApiLambdaCrudGenericoAOT` (nombre del ejecutable) |
| Memory | `2048 MB` |
| Timeout | `30 segundos` |
| Architecture | `x86_64` |
| Package | `Zip` · `--self-contained true` |

---

## Arquitectura

```
ApiLambdaCrudGenericoAOT/
├── Function.cs                         ← Entry point AOT, HTTP handling, ramas CRUD
├── Aplicacion/
│   ├── DTO/Generico/  (RespuestaGeneral, DocumentoPaginado)
│   └── Interfaces/    (IDocumentRepository)
└── Infraestructura/
    ├── Repositorio/   (DocumentRepository — Find/Project/Aggregate/Update/Delete)
    ├── Seguridad/     (ValidadorPipeline — NUEVO)
    └── Utilidades/    (Utils — compresión gzip/zstd/brotli, Base64 url-safe, logs)

Dependencia de proyecto:
LambdasLogiGho.Infraestructura/Seguridad/Encripcion.AES/EncripcionAES.cs  ← AES-256-ECB
```
