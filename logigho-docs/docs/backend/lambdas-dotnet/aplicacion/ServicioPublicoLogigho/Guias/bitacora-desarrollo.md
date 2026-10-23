---
autor: Claude (chat de desarrollo) con Iker Acevedo
fecha_creacion: 2026-10-22
ultima_actualizacion: 2026-10-22
estado: vivo
---

# Bitácora de desarrollo — API pública de guías

Una entrada por paso cerrado. Cada una dice qué se hizo, por qué, qué se descartó, cómo se verificó y qué queda
pendiente. Código en la rama local `feature/api-publica-guias` del repo de lambdas
(`LambdasLogiGho.Aplicacion/ServicioPublicoLogigho/Guias/`); documentación en `docs/api-publica-guias`.
Nada de esto está publicado ni desplegado.

Comando de verificación usado en cada paso (build Release sin advertencias, tests, paquetes vulnerables y
empaquetado de las dos Lambdas con `dotnet lambda package`).

---

## 2026-10-22 · F1a-a · Adaptador de Interrapidísimo

**Commit de código:** `2c9941e` — `feat(guias): adaptador de Interrapidisimo para crear guias (F1a-a)`

### Qué se hizo

El worker ya sabe crear una guía en Interrapidísimo. Es la pieza que, hasta hoy, vivía dentro de
`CargaInterUseCase` en `ApiLambdaCrearGuia` mezclada con Mongo, S3 y reglas de negocio. Ahora está aislada detrás de
la interfaz `ICreadorGuia` (en Aplicación) y la implementa `CreadorGuiaInterrapidisimo` (en Infraestructura).

Para poder armar el request, la solicitud (`SolicitudGuia`) dejó de ser un cascarón y ahora guarda:

- `DatosGuia`: lo que declara el integrador (destinatario, dirección, medidas, valor declarado, tipo de entrega,
  productos). Solo la forma; las validaciones llegan con `reglas-negocio.md` (F1b).
- `ContextoEnvio`: lo que el servidor resuelve y nunca viene del body (nombre de tienda, sucursal remitente,
  descripción del tipo de entrega, ciudad, origen).
- `IdCargaPedido` e `IdPedidoInter`: los `_id` (formato ObjectId) con los que más adelante se insertarán los
  documentos en `CargaPedido` y `PedidosInter`.
- `RespuestaTransportadoraJson`, `PasosCompletados` y `UrlGuia`: para que, si algo falla después de crear la guía,
  el reintento termine solo lo que falta sin volver a llamar a la transportadora.

### Archivos principales

| Archivo | Para qué |
| --- | --- |
| `Dominio/Guias/DatosGuia.cs` | Datos de la guía y contexto resuelto por el servidor |
| `Dominio/Guias/IdsDocumento.cs` | Genera ids con formato ObjectId sin depender de Mongo |
| `Dominio/Guias/CostosTransportadora.cs` | Lee `valorFlete`, `valorServicioContraPago`, `valorSobreFlete` de la respuesta de Inter |
| `Dominio/Solicitudes/SolicitudGuia.cs`, `Paso.cs` | Máquina de estados ampliada (nueva transición `Creando → Reservada`) y pasos posteriores |
| `Aplicacion/Puertos/ICreadorGuia.cs` | Cuatro resultados posibles: Creada, Rechazada, Incierta, **NoEnviada** |
| `Aplicacion/Trazas/TrazaHttp.cs` | Request/response de la llamada, con credenciales enmascaradas (ADR-005) |
| `Infraestructura/Transportadoras/Interrapidisimo/*` | Opciones, cuerpo de la admisión, adaptador y registro con resiliencia |
| `Infraestructura/Persistencia/RepositorioSolicitudesMongo.cs` | Guarda los datos nuevos |
| `test/.../Transportadoras/CreadorGuiaInterrapidisimoTests.cs` | 33 pruebas contra un Inter simulado (WireMock.Net) |

### Cómo clasifica lo que responde Inter

| Qué pasa | Resultado | Por qué |
| --- | --- | --- |
| HTTP 2xx con `numeroPreenvio` legible | **Creada** | Hay número de guía |
| HTTP 2xx sin `numeroPreenvio` legible | **Incierta** | Inter dijo "ok" pero no sabemos qué creó |
| HTTP 4xx de negocio (400, 422…) | **Rechazada** | Inter dijo que no; el motivo (máx. 500 caracteres) se devuelve al integrador |
| HTTP 500/502/503/504/408 | **Incierta** | Inter pudo crear la guía antes de fallar. **No se reintenta** |
| Timeout (30 s por defecto) | **Incierta** | Igual que arriba |
| Error de red *después* de enviar | **Incierta** | No sabemos si Inter alcanzó a procesar |
| Conexión rechazada, DNS, TLS (antes de enviar) | **NoEnviada** | Es seguro reintentar; la pipeline reintenta 2 veces |
| HTTP 401/403/429 | **NoEnviada** | Credenciales de LogiGho mal o Inter limitando: no se creó nada |
| Circuito abierto (5 fallos en 30 s) | **NoEnviada** | Se deja de golpear a Inter un rato |

Cuando el resultado es **NoEnviada**, la solicitud vuelve de `Creando` a `Reservada` y el mensaje de SQS falla a
propósito para que se reentregue. Si se agotan las entregas, cae en la DLQ (que tiene alarma). Así una caída corta de
Inter no pierde guías ni las marca como fallidas.

### Decisiones y alternativas descartadas

- **Sin reintento ciego del POST.** Hoy `Generico.consumoGenerico` repite cualquier POST hasta 4 veces, incluida la
  creación (ADR-003). El nuevo adaptador solo reintenta si el request ni siquiera salió. Esa política está en el
  *handler de resiliencia* del `HttpClient`, no dentro del adaptador, y un test demuestra que un 500 genera exactamente
  una llamada.
- **No se manda el token del usuario a Inter** (hallazgo de seguridad en `hallazgos-lambdas-actuales.md`). Un test
  revisa que no salgan los headers `Token` ni `Authorization`.
- **Credenciales de Inter**: se leen de las mismas variables de entorno cifradas con AES que usan las lambdas actuales
  (`URL_SERVICIO_INTER`, `ID_CLIENTE`, `USER_AUTH`, `TOKEN_AUTH`, `SUCURSAL_GENERICA`). No se crean secretos nuevos.
- **Se replicó tal cual la partición del nombre** (`"Maria Jose Lopez Gomez"` → nombre `Maria Jose`, primer apellido
  `Jose`, sin segundo apellido). Es una rareza heredada; se deja igual para que Inter reciba lo mismo que hoy y se fija
  con tests. **Pregunta para Iker:** ¿queremos mejorar esa partición más adelante? Cambiaría lo que ve Inter.
- **`AplicaContrapago` siempre `false`**: v1 no tiene contraentrega.
- **IDs de documento generados en el Dominio**, no por Mongo: así insertar `CargaPedido`/`PedidosInter` con un `_id`
  conocido hace idempotente el reintento (clave duplicada = "ya estaba") sin crear índices nuevos sobre colecciones
  existentes. Descartado: índice único en `PedidosInter.Numeropreenvio` (tipo mixto string/número y posibles
  duplicados sin verificar, ver `colecciones.md`).
- **`HttpRequestError` decide si el request salió.** Solo `ConnectionError`, `NameResolutionError`,
  `SecureConnectionError` y `ProxyTunnelError` se consideran "antes de enviar". Riesgo conocido y aceptado: si el
  servidor cerrara la conexión justo durante el envío podría clasificarse como "antes"; en ese caso se reintenta una
  vez y, si Inter ya había creado la guía, habría un duplicado. Es el único hueco y está acotado a 2 reintentos con
  error de *conexión*, no de respuesta.

### Cómo se verificó

- 91 tests en verde (38 del esqueleto + 53 nuevos), incluyendo WireMock.Net con el `HttpClient` y la pipeline reales:
  éxito, número como texto, cuerpo idéntico al actual, sucursal genérica, 400, motivo truncado, 5xx y 408 sin
  reintento, 401/403/429, timeout sin reintento, conexión rechazada real y simulada (3 intentos), error de red tras
  enviar (1 intento), circuito abierto (la sexta llamada no sale), transportadora equivocada.
- `dotnet build -c Release`: 0 advertencias, 0 errores.
- `dotnet list package --vulnerable --include-transitive`: sin paquetes vulnerables.
- `dotnet lambda package` genera `ApiLambdaGuias.zip` (9,6 MB) y `ApiLambdaGuiasWorker.zip` (9,9 MB).
- Paquetes nuevos (todos publicados hace más de 2 semanas): `Microsoft.Extensions.Http` 10.0.12 (08-sep),
  `Microsoft.Extensions.Http.Resilience` 10.10.0 (09-sep), `WireMock.Net` 2.18.0 (22-sep, solo tests).

### Pendiente / siguiente

- **F1a-b:** escritores de `CargaPedido` y `PedidosInter` con el mismo documento de hoy.
- Falta conectar el caso de uso para que, tras `Creada`, ejecute los pasos posteriores.
- **Para Iker:** el nombre de empresa, la sucursal y los catálogos (`CiudadesInter`, `TipoEntregaInter`, `Tienda`)
  los resolverá el endpoint en F1b; por ahora los tests usan datos ficticios.

---

## 2026-10-22 · F1a-b · Escritores de `CargaPedido` y `PedidosInter`

**Commit de código:** `664012a` — `feat(guias): escritores de CargaPedido y PedidosInter con el documento actual (F1a-b)`

### Qué se hizo

Cuando el worker crea una guía, ahora deja las mismas huellas en Mongo que dejan hoy `ApiLambdaCrearGuia` y
`ApiLambdaCargarGuias`, para que liquidaciones, sincronizadores, inventario y Angular no noten la diferencia:

1. **Antes de llamar a Inter** inserta la fila en `CargaPedido` con `Estado = "Pendiente"` y `Origen = "API"`.
2. **Si Inter crea la guía**, la fila pasa a `Cargado` con `NumeroPreenvio`, `Total Flete`, `Total Servicio`,
   `Total Sobreflete`, `Cantidad Total Productos` y `Trayecto` vacío; y se inserta el documento de `PedidosInter`
   con `Estado = "Creado"`.
3. **Si Inter la rechaza**, la fila queda `Rechazado` con el campo `ERROR`, como hoy.
4. **Si la respuesta es incierta** (timeout), la fila se queda en `Pendiente`: todavía no sabemos si la guía existe, así
   que ni se carga ni se rechaza. Lo resuelve el reconciliador (F1a-g).

### Cómo se garantiza "el mismo documento que hoy"

`DocumentosPedido` (Infraestructura) arma los documentos con el mismo algoritmo de `CargaInterUseCase`
(`ApiLambdaCrearGuia:186-253`): toma la fila, le cambia estado y costos, le quita `_id` y `ERROR`, pasa las claves por
`ConvertToPascalCase` (minúsculas → título por palabra → sin espacios) y agrega los campos de seguimiento vacíos
(`Estado Logigho`, `Fecha Entrega`, `Pago`, …).

La prueba que lo protege **ejecuta una copia literal del algoritmo actual** (con `JToken` de Newtonsoft) sobre la misma
fila y compara el resultado con el nuestro, campo por campo, tipo por tipo y en el mismo orden, en 10 casos
(nombres de 1 a 4 palabras, sin seguro, etiqueta mediana, varios productos, sin productos, identificación no
numérica, medidas decimales, teléfono mayor que int32, acentos y comillas, valor alto) más 4 formas distintas de la
respuesta de Inter (número como texto, costos nulos, sin costos). Además hay una **foto de referencia** del documento
de `PedidosInter` completo: si un nombre de campo, un tipo o un valor cambia, la prueba falla y obliga a decidirlo a
propósito.

### Idempotencia sin índices nuevos

`CargaPedido` y `PedidosInter` las usan otros sistemas; crear un índice único sobre ellas no es algo que se haga de
pasada (`PedidosInter.Numeropreenvio` tiene tipo mixto y posibles duplicados). En vez de eso, cada solicitud lleva dos
`_id` (formato ObjectId) generados desde el principio: uno para su fila de `CargaPedido` y otro para su documento de
`PedidosInter`. Si un reintento repite el insert, Mongo responde "clave duplicada" y se da por hecho que ya estaba.
Las actualizaciones son `$set` de los mismos valores, repetibles sin efecto.

### Qué pasa si algo falla a mitad de camino

La solicitud recuerda qué pasos ya hizo (`CargaInsertada`, `CargaActualizada`, `PedidoInsertado`, `CargaRechazada`).
Si el insert en `PedidosInter` falla, el mensaje de SQS falla, vuelve a la cola y el siguiente intento **solo repite ese
paso**; nunca vuelve a llamar a Inter. Hoy ese caso es el hallazgo "insert en `PedidosInter` fallido solo se registra en
log": la guía queda creada y el pedido huérfano, sin aviso. Aquí el fallo llega hasta la DLQ, que tiene alarma.

### Decisiones, supuestos y preguntas para Iker

| Tema | Lo que se hizo | Por qué / qué falta confirmar |
| --- | --- | --- |
| Fila de `CargaPedido` | Mismos campos y tipos que la fila de una guía manual en Angular (`modal-creacion-pedidos`) | `reglas-negocio.md` aún no existe; si la lista de campos del API cambia, cambia aquí |
| `Origen` | `"API"` | Lo dice el diseño; el código actual solo distingue `"CRM"` (va a `CargaPedidoCrm`), así que `"API"` va a `CargaPedido` |
| `IdCarga` | `"-1"` (como la guía manual) | **Confirmar**: ¿las guías de la API deben tener otro `IdCarga`? |
| `Idtienda` | Entero | **Confirmar con datos reales**: en `PedidosInter` puede haber string o int (hallazgo de `cupo.md`) |
| `ASESOR` | `"Logigho"` (valor por defecto de Angular) | **Confirmar**: podría ser útil guardar el cliente API, pero cambia el documento |
| `CODIGO OFICINA`, `CODIGO REGIONAL`, `VALOR BASE PRODUCTO` | `1`, `3`, `10000`: constantes que hoy manda Angular | Se copian tal cual |
| `Total recaudo` | Queda igual al **valor declarado** aunque no haya contraentrega | **Hallazgo**: así lo hace hoy `CargaInterUseCase` (línea 199). `cupo.md` supone recaudo 0 para guías sin COD; hay que decidir si la liquidación lo lee de `PedidosInter` y cómo lo trata |
| Fecha | Hora de Colombia (UTC−5) fija entre reintentos | La solicitud guarda `RecibidaEn` y `CreadaEn` para que un reintento escriba la misma fecha |
| `Cantidad Total Productos` | Suma de todas las cantidades | `ApiLambdaCrearGuia` suma solo hasta 12 productos y `ApiLambdaCargarGuias` hasta 18; la API limitará por regla (F1b) |
| `UrlGuia` | Se escribe en un paso aparte (etiqueta, F1a-c) | Hoy `CargarGuias` lo inserta con el documento y `CrearGuia` lo actualiza después; el resultado final es igual |
| Forma de pago | `CREDITO` para Inter; `CONTADO` para Envía (invertida) | Regla de `reglas-modalidad-pago.ts`; sin contraentrega |

### Cómo se verificó

- 146 tests en verde, 10 de ellos de **integración contra el Mongo local** (servicio `mongod` ya instalado en el equipo;
  cada prueba usa su propia base `LogighoTest_*` que se borra, y solo corre si el servidor es `localhost`):
  insert idempotente y con 10 llamadas simultáneas, update, `Rechazado` con motivo, `UrlGuia` en el documento correcto,
  flujo completo con repositorios reales (una sola guía aunque el mensaje llegue dos veces) e ida y vuelta de la
  solicitud completa.
- Build Release sin advertencias, sin paquetes vulnerables, las dos Lambdas empaquetan.
- CI (`pipelines/servicio-publico`): ahora levanta `mongo:7` y define `MONGO_TEST_URI`, igual que el CI de novedades.
  Sin Mongo, localmente, esas pruebas se **omiten** y se ven como omitidas, no como aprobadas.
- Se encontró y corrigió una prueba intermitente del paso anterior (timeout de 5 s demasiado justo en el primer
  request en frío); ahora el margen es de 30 s y se probó en 3 compilaciones limpias seguidas.
- `Newtonsoft.Json` 13.0.4 (16-sep-2025) entra solo en el proyecto de tests, únicamente para ejecutar la copia del
  algoritmo actual.

### Pendiente / siguiente

- **F1a-c:** etiqueta de Inter a S3 (mismos buckets y claves), aviso a inventario y trazas a S3, como pasos
  reintentables. Hoy `PasosPendientes` solo incluye los dos de Mongo.
- Una solicitud que se queda en `Creando` porque la Lambda murió antes de recibir respuesta no avanza sola: la recoge
  el reconciliador (F1a-g).
- Docker Desktop no arranca en este equipo (WSL1 no es compatible con la configuración actual); no hizo falta porque ya
  hay un `mongod` local. Si algún día se necesita Docker, hay que habilitar WSL2.

---

## 2026-10-22 · F1a-c · Etiqueta de Inter en S3, aviso a inventario y trazas

**Commit de código:** `feaaf10` — `feat(guias): etiqueta de Inter a S3, aviso a inventario y trazas en S3 (F1a-c)`

### Qué se hizo

Después de que Inter crea la guía y de que quedan `CargaPedido` y `PedidosInter`, el worker completa tres cosas más,
cada una como un **paso que se puede repetir sin duplicar nada**:

1. **Etiqueta.** Pide el PDF a Inter (`Admision/ObtenerBase64PdfPreGuiaFormatoPeq/{n}` para sticker,
   `.../ObtenerBase64PdfPreGuia/{n}` para mediana), lo valida, lo publica en S3 y escribe `UrlGuia` en `PedidosInter`.
2. **Inventario.** `POST {URL_SERVICIO_AWS}/actualizaInventarioLotes` con `{"Guias":["<número>"]}`.
3. **Trazas.** Guarda en S3 el request y la response exactos de cada llamada a Inter.

Orden de los pasos de una guía creada: `CargaActualizada → PedidoInsertado → EtiquetaSubida → InventarioAvisado`.
El inventario va al final porque lee la guía de `PedidosInter`.

### Etiqueta: mismo destino que hoy (ADR-004)

Se verificó contra `ApiLambdaPutObjectAOT` y contra AWS (solo lectura) que los buckets existen
(`bucket-guias-inter-prod`, `bucket-guias-pdf-prod`) y que el dominio público es el de CloudFront configurado en esa
lambda (`URL_CLOUDFRONT`). El worker escribe con el SDK de S3, con el mismo formato:

| Archivo | Bucket | Clave | Contenido |
| --- | --- | --- | --- |
| Texto | `BUCKET_GUIAS_INTER` | `{numeroGuia}.txt` | El PDF en base64 como texto (igual que hoy) |
| PDF | `BUCKET_GUIAS_PDF` | `guias-pdf/inter/{id}.pdf` | Bytes del PDF, `application/pdf`, `inline` |
| `UrlGuia` | — | — | `URL_CLOUDFRONT` + clave del PDF |

La prueba levanta un S3 simulado (WireMock) y usa el **SDK real**, así comprueba rutas, headers y contenido tal como
saldrían a S3. El `{id}` del PDF es **aleatorio y se guarda en la solicitud antes de subir el archivo**: si el paso se
repite escribe en la misma clave (no deja archivos huérfanos) y la URL sigue siendo impredecible. Importa porque el PDF
es público y contiene datos personales: una clave derivada del número de guía se podría adivinar.

### Trazas (ADR-005)

`trazas/{yyyy}/{MM}/{dd}/{numeroGuia}/{solicitudId}-{paso}.json.gz` en el bucket `BUCKET_TRAZAS_GUIAS`, JSON
comprimido con solicitud, cliente, tienda, request y response. Las credenciales de Inter van como `***`. En Mongo solo se
anota la clave, el código HTTP y la latencia. Se guardan la admisión (incluso si Inter la rechaza o queda incierta) y la
petición de etiqueta; esta última **sin el PDF**, que ya está en el bucket. Si S3 falla al guardar una traza, la guía
sigue su camino y se deja un aviso en el log: la traza sirve a soporte, no al flujo.

### Qué pasa si algo falla

| Falla | Resultado |
| --- | --- |
| Inter devuelve PDF vacío, concatenado, con su mensaje de error o cortado | Se reintenta hasta 3 veces; si ninguno es válido, el mensaje de SQS falla y vuelve a la cola. La guía queda `Creada` y **no se rechaza** (hoy la carga masiva marca la fila `Rechazado` aunque la guía ya existe) |
| S3 rechaza la escritura | Igual: el paso se repite desde la cola |
| El inventario responde error | Reintento corto con espera (2 veces); si persiste, el mensaje vuelve a la cola y solo se repite ese paso |

### Decisiones y alternativas descartadas

- **No se usa `/putObject`.** Esa ruta no tiene authorizer ni API key (hallazgo de ADR-004) y exige un token de usuario
  que un cliente de máquina no tiene.
- **Validación de PDF sin PDFsharp.** El método actual (`PdfUtils.IsValidPdf`) abre el PDF con PDFsharp para contar
  páginas. PDFsharp 6.2.4 trae `System.Security.Cryptography.Pkcs` 8.0.1 con **dos vulnerabilidades conocidas**
  (GHSA-846f-f764-76mx, alta; GHSA-2fmv-rv2c-59x7, moderada) y los parches salieron el 13-oct-2026, así que todavía no
  cumplen la regla de 2 semanas. En su lugar se revisa: base64 decodificable, cabecera `%PDF`, que no haya dos cabeceras
  (PDFs pegados), que no contenga el mensaje de error de impresión de Inter y que termine con `%%EOF`. Es más ligera
  que abrir el archivo. Se puede volver a PDFsharp después del 27-oct-2026.
- **Ocultar al remitente "LOGIGHO" en el sticker: pendiente de decisión.** Consulté la configuración de las lambdas
  en producción (solo lectura): `ApiLambdaCargarGuias` tiene `OCULTAR_REMITENTE_ETIQUETA=true` con
  `TEXTOS_OCULTAR_ETIQUETA=LOGIGHO,LOGIGHO S.A.S` y reescribe el PDF con iText; `ApiLambdaCrearGuia` y
  `ApiLambdaCrearEtiquetaManual` **no** lo hacen. `MODIFICACION_ETIQUETA` está en `false` en las tres. La API publica
  hoy la etiqueta tal como la entrega Inter (`TransformadorEtiquetaIdentidad`), igual que la creación manual.
  **Pregunta para Iker:** ¿las etiquetas de la API deben ocultar "LOGIGHO" como la carga masiva? Si sí, hay que portar
  `EditorContenidoPdf` (iText, ~350 líneas y una dependencia AGPL) detrás de la misma interfaz
  `ITransformadorEtiqueta`; no cambia nada más del flujo.
- **Auditoría de direcciones de etiqueta** (`AuditoriaEtiquetasInter`): en producción `AUDITORIA_ETIQUETA` no está
  definida, o sea apagada. No se porta.
- **Inventario con `Guias`, no `Idcarga`.** `ApiLambdaInventarioCarga` acepta una lista de guías; es lo que pide el
  diseño. CargarGuias envía `Idcarga` y sin esperar respuesta (la Lambda puede congelarse antes de que salga la
  llamada); aquí se espera la respuesta.
- **Configuración.** Se reusan los nombres de variable de hoy (`BUCKET_GUIAS_PDF`, `URL_CLOUDFRONT`,
  `URL_SERVICIO_AWS`) y se crean dos nuevas que debe definir Terraform: `BUCKET_GUIAS_INTER` y `BUCKET_TRAZAS_GUIAS`. Hoy
  el bucket de texto está escrito en el código con el sufijo `-prod` incluso en PreProd (hallazgo de
  `hallazgos-lambdas-actuales.md`); aquí es configurable por entorno.

### Cómo se verificó

- 191 tests en verde: S3 simulado con el SDK real (rutas, content-type, `inline`, comillas y espacios en el base64,
  repetir la publicación, base64 inválido, S3 con error 403); trazas gzip (clave, JSON, credenciales ocultas, carpeta
  `sin-guia`); inventario (cuerpo `{"Guias":[...]}`, sin header `Authorization`, reintento ante 503, error 400 sube);
  etiqueta de Inter (ruta por tipo, base64 con y sin comillas, primer PDF inválido y segundo bueno, tres inválidos,
  error HTTP persistente, sin credenciales de usuario ni PDF en la traza); validador de PDF; caso de uso (orden de pasos,
  el reintento no repite lo ya hecho, fallo de traza no frena la guía); y el **contenedor real del worker** con las
  variables de entorno cifradas con AES.
- Build Release sin advertencias, sin paquetes vulnerables, las dos Lambdas empaquetan (ahora ~12 MB cada una).
- Paquete nuevo: `AWSSDK.S3` 4.0.104.1 (1-oct-2026, más de 2 semanas; la 4.0.104.2 salió el 8-oct y se dejó fuera por
  margen).

### Pendiente / siguiente

- **Terraform** (paso 4 del plan): crear el bucket de trazas con lifecycle de 90 días y definir las dos variables nuevas.
- Decidir lo de ocultar el remitente (arriba).
- **F1a-d:** `ApiClientes` y autorización. **F1a-e:** idempotencia en `ApiSolicitudesGuia`. **F1a-f:** cupo.
  **F1a-g:** reconciliador (también recoge las solicitudes que se quedan en `Creando` o `Incierta`).

---

## 2026-10-22 · F1a-d · `ApiClientes` y autorización

**Commit de código:** `6f7debc` — `feat(guias): ApiClientes y autorizacion por claims, scopes e IP (F1a-d)`

### Qué se hizo

Cada llamada a la API ahora pasa por una autorización en dos capas, como pide [ADR-002](adr-002-autenticacion.md):

1. **API Gateway** (aún por configurar en Terraform) valida el token de Cognito y el scope del método.
2. **El Lambda** vuelve a comprobar, usando solo los *claims* que el gateway ya validó:
   - `client_id` (qué integrador es) y `scope` (qué permisos trae el token) de `HttpContext.User`;
   - la IP de origen que ve el gateway;
   - el perfil del cliente en `ApiClientes` (con caché de 30 s).

Se rechaza, **en este orden**: sin identidad (401 `NO_AUTENTICADO`) → cliente sin perfil (`CLIENTE_NO_REGISTRADO`) →
cliente no activo (`CLIENTE_BLOQUEADO`) → falta el scope (`SCOPE_INSUFICIENTE`) → IP fuera de la lista
(`IP_NO_PERMITIDA`). Un cliente bloqueado siempre ve `CLIENTE_BLOQUEADO`, aunque además le falte el scope.
Todos son `ProblemDetails` con `codigo` y `traceId`. El mensaje nunca revela datos de otros clientes ni la lista de IPs.

El perfil también expone las comprobaciones por **tienda, transportadora y producto** (`PermiteEnvio`), que usará el
endpoint de crear guías (F1b). Primeros endpoints protegidos, a modo de uso real: `GET /v1/tiendas` y
`GET /v1/transportadoras` (scope `guias:leer`), que devuelven solo lo que el perfil del cliente permite.

### Por qué por claims y no leyendo el token

En `/pedidos` hoy la lambda lee el header `Token` y decodifica el JWT **sin verificar la firma** mientras el gateway
valida otro header (`Authorization`): quien mande un JWT inventado en `Token` podría hacerse pasar por otro. La API nueva
no lee ningún header de token. Dos pruebas lo fijan:

- Un JWT falso con el `client_id` de otro cliente en `Authorization` y `Token` **no** suplanta a nadie (responde 401).
- Una prueba usa la **librería real** `Amazon.Lambda.AspNetCoreServer` con un evento REST de API Gateway y comprueba que
  los claims del authorizer llegan a `HttpContext.User` y que la IP (`sourceIp`) llega a la conexión. Si una actualización
  de la librería cambiara eso, la prueba avisa antes de que la seguridad deje de funcionar.

### Decisiones y supuestos

| Tema | Decisión | Por qué / qué confirmar |
| --- | --- | --- |
| Estado desconocido | Se trata como **bloqueado** | Fail-closed: un error de tipeo en `ApiClientes` no abre la puerta |
| Lista de productos vacía | **Ningún producto permitido**; una guía sin productos no consulta la lista | **Confirmar con Iker**: ¿vacío debería significar "todos los de la tienda"? Se eligió lo más restrictivo |
| Lista de IPs vacía | Cualquier origen | Así lo dice el diseño ("lista opcional de IPs permitidas") |
| Entrada de IP ilegible | Se ignora | Con lista llena, lo ilegible no abre la puerta |
| Scopes | Se comparan por nombre, sin el prefijo del resource server | Cognito entrega `logigho/guias:crear`; así el código no depende del identificador de cada entorno. Riesgo: otro resource server con el mismo nombre de scope. El pool solo tendrá el nuestro |
| Caché | 30 s, incluida la de clientes inexistentes | Un bloqueo se nota en menos de 30 s; sin esto, un cliente sin perfil provocaría una lectura de Mongo por request |
| Códigos nuevos | `NO_AUTENTICADO`, `CLIENTE_NO_REGISTRADO`, `SCOPE_INSUFICIENTE`, `IP_NO_PERMITIDA` | Agregados a [Endpoints](endpoints.md) |
| Forma de `ApiClientes` | Documentada en [Colecciones](colecciones.md) | El código solo lee; la administración (alta, bloqueo, rotación de secreto) es otra fase |

### Un hallazgo de diseño (de las pruebas)

Al registrar los casos de uso en la API, la validación de dependencias de ASP.NET (que corre en las pruebas) falló
porque la API intentaba construir el caso de uso **del worker**, que necesita la transportadora. Se separó el registro:
`AddAplicacionGuias` (API) y `AddProcesamientoGuias` (worker). Un error así en producción habría tumbado el arranque
de la Lambda; las pruebas lo atraparon antes.

### Cómo se verificó

- 269 tests en verde (78 nuevos): permisos por tienda/transportadora/producto, listas de IP (exacta, CIDR, IPv6, IPv4
  mapeada, entradas ilegibles), lectura de scopes, caso de uso con cada rechazo y su orden, caché con reloj manual
  (vence a los 30 s, recuerda inexistentes), lectura de `ApiClientes` en el Mongo local (campos extra ignorados, estado
  y transportadoras desconocidos), la API completa con claims simulados (401, cada 403, listados) y el contrato con la
  librería de Lambda.
- Build Release sin advertencias, sin paquetes vulnerables, las dos Lambdas empaquetan.

### Pendiente / siguiente

- **F1a-e:** idempotencia en `ApiSolicitudesGuia` e índices desde código (incluye `ux_ApiClientes_ClienteId`).
- Las rutas de administración de clientes y la rotación de secretos no están (fase de administración).
- En Terraform (paso 4): authorizer de Cognito con scopes por método y usage plan con API key.

---

## 2026-10-22 · F1a-e · Idempotencia e índices desde código

**Commit de código:** `49164fe` — `feat(guias): idempotencia por Idempotency-Key e indices creados desde codigo (F1a-e)`

### Qué se hizo

Es la pieza que impide que la misma guía se cree dos veces cuando el integrador reintenta (por ejemplo, porque su
conexión se cortó justo antes de recibir la respuesta). Cumple el punto 1 de [ADR-003](adr-003-idempotencia.md):

- Cada `POST` trae un header `Idempotency-Key` (el integrador usa, por ejemplo, el id de su pedido).
- Al llegar, se intenta **insertar** la solicitud en `ApiSolicitudesGuia`. La base tiene un **índice único**
  `{ClienteId, IdempotencyKey}`: si la clave ya existe para ese cliente, el insert se rechaza.
- Si ya existía, se compara la **huella del body** (`HashCuerpo`) con la de la solicitud guardada:

| Situación | Resultado | Qué ve el integrador |
| --- | --- | --- |
| Primera vez que llega esa key | **Nueva** | Se procesa la solicitud |
| Misma key, mismo body, ya hay resultado (guía creada, fallo definitivo o incierta) | **Repetida** | Se devuelve lo guardado, sin crear nada |
| Misma key, mismo body, todavía sin resultado (`RECIBIDA`, `RESERVADA`, `CREANDO`) | **EnCurso** | `409 SOLICITUD_EN_CURSO`: espere y consulte |
| Misma key, **otro** body | **Conflicto** | `422 IDEMPOTENCY_CONFLICTO` |

La key es **por cliente**: dos integradores distintos pueden usar la misma sin chocar. Una solicitud `INCIERTA` cuenta
como *resultado* (Repetida) y no como "en curso", para que el integrador vea que está incierta en vez de esperar
indefinidamente.

### Por qué la garantía está en el índice y no en el código

Si la API revisara "¿ya existe esta key?" y luego insertara, dos requests simultáneos podrían pasar la revisión a la vez
(más aún con varios contenedores de Lambda). El índice único hace que **la base decida atómicamente** cuál gana. Una
prueba lo demuestra con 50 requests en paralelo, cada uno con su propio cliente de Mongo (como si fueran contenedores
distintos): exactamente 1 queda como Nueva, 49 ven "en curso" y en la colección hay un solo documento.

### La huella del body (`HashCuerpo`)

SHA-256 del JSON **canónico**: propiedades ordenadas, sin espacios y con los números normalizados. Así, si la librería
del integrador reordena campos o escribe `1.0` en lugar de `1` en el reintento, no se confunde con un conflicto. Un
cambio real (otro valor, un campo de más o de menos, otro orden en un arreglo, mayúsculas, un espacio al final del texto)
sí cambia la huella. Los vectores de prueba se calcularon con un SHA-256 independiente (Python).

### Índices desde código

`GarantiaIndices` crea los índices de las colecciones `Api*` antes de la primera inserción de cada contenedor, tanto en
PreProd (Atlas) como en producción (DocumentDB):

| Colección | Índice | Para qué |
| --- | --- | --- |
| `ApiSolicitudesGuia` | `ux_ApiSolicitudesGuia_Cliente_Key` (único) | Idempotencia |
| `ApiSolicitudesGuia` | `ix_ApiSolicitudesGuia_NumeroGuia` | Consultar por número de guía |
| `ApiSolicitudesGuia` | `ix_ApiSolicitudesGuia_Estado_Fecha` | Reconciliador (buscar solicitudes atascadas) |
| `ApiClientes` | `ux_ApiClientes_ClienteId` (único) | Un perfil por cliente |

Reglas: si no se pueden crear, **la inserción falla** (sin índice único no hay idempotencia, así que es mejor no seguir);
crear un índice que ya existe no hace nada, por lo que muchos arranques en frío a la vez no se estorban (también
probado); solo se usan opciones que DocumentDB soporta (índices simples y compuestos, únicos y TTL; nada parcial ni de
texto); y **solo colecciones propias** (`Api*`). Los índices de colecciones que usan otros sistemas
(`PedidosInter`, `LiquidacionesLogigho`…) quedan para crearse aparte, en horario valle, tal como pide `cupo.md` §8.8; una
prueba comprueba que esta lista nunca los incluya.

### Lo que no cubre este paso (y por qué importa)

- **Solicitud atascada antes de encolar.** Si la API inserta la solicitud (`RECIBIDA`) y se cae antes de reservar cupo o
  de encolar el mensaje, esa key queda "en curso" para siempre. No es posible resolverlo en el registro; lo resuelve el
  reconciliador (F1a-g), que debe reencolar o dar por fallidas las solicitudes con más de N minutos sin avanzar. El índice
  `Estado + FechaActualizacion` existe justamente para eso.
- **El endpoint** `POST /v1/guias` que usa todo esto llega en F1b (necesita las reglas campo por campo).

### Decisiones y supuestos

| Tema | Decisión | Por qué |
| --- | --- | --- |
| `Incierta` | Cuenta como resultado (Repetida) | El integrador debe poder ver "incierta" en lugar de esperar |
| Body | Se hashea el JSON recibido, canonicalizado | Tolera reordenamientos del cliente sin perder detección de cambios reales |
| Key | Por cliente, no global | Dos integradores pueden usar el mismo id de pedido |
| Fallo al crear índices | La inserción falla | Mejor un error visible que perder la garantía en silencio |

### Cómo se verificó

- 314 tests en verde (45 nuevos): huella (vectores, orden, espacios, números, cambios reales, unicode, JSON inválido),
  caso de uso por cada estado de la solicitud, 20 requests simultáneos sobre el repositorio en memoria, y contra el
  **Mongo local**: nombres y opciones de los índices, creación repetida y simultánea, insert duplicado, clave por
  cliente, 50 requests en paralelo con clientes de Mongo distintos, 20 con bodies distintos (1 ganadora, 9 "en curso",
  10 "conflicto"), solicitud terminada que se devuelve con sus datos, y que guardar no rompa el índice.
- Build Release sin advertencias, sin paquetes vulnerables, las dos Lambdas empaquetan.

### Pendiente / siguiente

- **F1a-f:** cupo (reserva atómica, ledger `ApiMovimientosCupo`, fail-closed) con pruebas de concurrencia en Mongo local.
- **F1a-g:** reconciliador (incluye las solicitudes atascadas de arriba).
