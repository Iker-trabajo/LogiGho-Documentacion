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
