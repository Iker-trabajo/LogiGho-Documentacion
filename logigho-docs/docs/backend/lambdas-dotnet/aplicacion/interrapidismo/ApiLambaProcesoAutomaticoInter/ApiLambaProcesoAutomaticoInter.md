## Autor: Iker Acevedo

Fecha creacion: 2026-08-20

Estado: produccion

## Lambda: ApiLambdaProcesoAutomaticoInter

**Accionador:** API Gateway (llamado periódicamente; cada ejecución reprocesa el lote completo de pedidos Inter activos, no un solo pedido)

**AOT:** No

---

## ¿Qué hace?

Es la lambda de sincronización periódica de pedidos Interrapidísimo. En cada corrida:

1. Trae de MongoDB **todos** los pedidos Inter que todavía no llegaron a un estado terminal (`ObtenerPedidosFiltradosAsync`, colección `PedidosInter`).
2. Consulta a la API de Interrapidísimo el estado actual de cada guía (`ConsultarEstadosGuiasCliente`).
3. Resuelve el `Estado` final de cada pedido en base al historial de eventos que reporta Inter (`estadosGuia`), cruzando además contra `ConciliacionPagos` (pagos) y `TarifasInter` (trayecto).
4. Actualiza `PedidosInter` en bulk con el estado recalculado.
5. **Auditoría de cambio de entrega a oficina** (esta lambda, desde 2026-07): detecta cuándo Inter cambió unilateralmente una guía que nació para **entrega en dirección** a **"Reclame en oficina"**, y por separado, cuándo esa guía finalmente se resolvió (entrega o devolución). Ambas cosas se guardan en una colección aparte, `AuditoriaCambioEntregaInter`, sin tocar `PedidosInter`.

El punto 5 es la funcionalidad construida en esta iteración y es el foco de este documento — el resto (puntos 1-4) es el comportamiento preexistente de la lambda.

---

## Accionador

| Método | Ruta | Autenticacion |
| ------ | ---- | ---- |
| `POST` | API Gateway — `ApiLambdaProcesoAutomaticoInter` | Bearer token (Cognito), opcional — se usa para autenticar las llamadas internas a `/metodoGenerico` y a la API de Inter, no filtra qué pedidos procesa |

No recibe body con parámetros — cada corrida procesa el universo completo de pedidos activos que trae `ObtenerPedidosFiltradosAsync`.

---

## Response

```json
{}
```

`200 OK` sin body significativo si la corrida termina. Los fallos de auditoría (puntos 5) **nunca** devuelven error — están en `try/catch` propios que solo loguean, para no tumbar la actualización principal de `PedidosInter` (puntos 1-4), que es la función crítica de la lambda.

| Código | Cuándo |
| ------ | ------ |
| `500` | Excepción no controlada fuera de los bloques protegidos (ej. falla total consultando la API de Inter) |

---

## Flujo interno

```
FunctionHandler (Function.cs)
  -> Carga en paralelo: ConciliacionPagos, Productos, TarifasInter, EstadoGuiaLogighoNew, PedidosInter filtrados
  -> Kill switch: AUDITORIA_CAMBIO_ENTREGA (por defecto activo, "false" lo apaga)
  -> Crea 2 ConcurrentDictionary<string, BsonDocument>: cambiosEntrega, resolucionDesenlace
  -> Parallel.ForEachAsync (lotes de 15, 5 en paralelo)
     -> CargaInterNegocioUseCase.DepuraCargaInter (por lote)
        -> Resuelve Estado final del pedido (switch sobre estadosGuia + pagos + Archivada→Error)
        -> [AUDITORÍA 1] DetectorCambioEntrega.Califica(pedido)
             ¿Estado == "Reclame en oficina" Y TipoEntrega == 1 (entrega en dirección)?
             -> cambiosEntrega[NumeroGuia] = ConstruirDocumento(...)
        -> [AUDITORÍA 2] DetectorCambioEntrega.CalificaResolucion(pedido)
             ¿Ya se resolvió (Entrega/Devolución) Y TipoEntrega == 1?
             -> resolucionDesenlace[NumeroGuia] = ConstruirDocumentoResolucion(...)
        -> Acumula el pedido para el bulk update de PedidosInter
  -> BulkUpsertDocumentsAsync("PedidosInter", ...)          [actualización principal]
  -> try { UpsertCambiosEntregaAsync(cambiosEntrega) }       [auditoría 1, upsert:true]
  -> try { ActualizarDesenlaceAsync(resolucionDesenlace) }   [auditoría 2, upsert:false]
  -> Logs: cuántas guías se detectaron vs cuántas se escribieron, en cada auditoría
```

Ninguna de las 2 auditorías agrega round-trips dentro del loop paralelo — ambas se acumulan en memoria y se escriben **una sola vez** en bulk, después de procesar todos los lotes.

---

## Auditoría de cambio de entrega a oficina (`AuditoriaCambioEntregaInter`)

### El problema que resuelve

Interrapidísimo a veces cambia unilateralmente una guía que se creó para **entrega en dirección** (`TipoEntrega = 1`) a **"Reclame en oficina"**, sin que el pedido se haya originado así. Antes de esta feature no había forma de saber cuántas guías pasan esto, ni si finalmente se resuelven como entrega o como devolución — la lambda simplemente sobrescribía el `Estado` en `PedidosInter` sin dejar rastro del historial.

### Diseño: 2 detecciones independientes, sobre la misma colección

Ver [`DetectorCambioEntrega.cs`](#clase-detectorcambioentregacs) para el detalle de cada regla. En resumen:

**1. Detección de la guía en oficina** (feature original, jul-2026): cuando una guía con `TipoEntrega = 1` llega a `Estado == "Reclame en oficina"`, se **inserta o actualiza** (`upsert: true`) un documento en `AuditoriaCambioEntregaInter` con todos los datos denormalizados del pedido en ese momento.

**2. Detección del desenlace final** (esta iteración, ago-2026): en **cada** corrida, para **cualquier** pedido que ya se resolvió (`Entrega` o `Devolucion`, ver regla más abajo) y tiene `TipoEntrega = 1`, se intenta escribir `EstadoFinal` + `FechaResolucion` sobre el mismo documento — pero con `upsert: false`. Si la guía nunca pasó por la detección 1 (no existe el documento), el update no hace nada: **nunca se crean documentos huérfanos** desde la detección 2.

Esto es intencional: la detección 2 no necesita saber en memoria si una guía "es de las auditadas" — se lo delega a Mongo con el filtro `NumeroGuia` + `upsert:false`. Así, una guía que se resolvió hace tiempo (y ya no vuelve a pasar por el loop porque `Pagada`/`Devolución ratificada` son estados terminales excluidos de `ObtenerPedidosFiltradosAsync`) puede seguir recibiendo la actualización de desenlace mientras siga entrando al batch por cualquier otro motivo — y si nunca vuelve a entrar, se cubre con el backfill único (ver más abajo).

### Regla de clasificación del desenlace

Reutiliza **tal cual** la regla de negocio ya probada en `DepurarLiquidacionUseCase.cs` (`ApiLambdaLiquidacionesLogighoAOT`) — es la fuente de verdad existente en el repo para "¿este pedido fue entrega o devolución?":

```csharp
esEntrega    = !string.IsNullOrEmpty(pedido["Fecha Entrega"])
               && new[] { "Entregada", "Digitalizada", "Archivada", "Pagada" }.Contains(estado);

esDevolucion = !esEntrega
               && (!string.IsNullOrEmpty(pedido["Fecha Devolucion"]) || estado == "Indemnizacion");
```

`Indemnizacion` cuenta como devolución aunque no siempre traiga `Fecha Devolucion` seteada (depende de si el historial de la guía pasó antes por un estado `Devoluci*`) — se agregó como condición explícita, acordado con el negocio.

Si ninguna de las 2 aplica, el pedido sigue "pendiente / en oficina" — no se escribe nada, el campo queda ausente (el front lo interpreta como pendiente).

### Clase `DetectorCambioEntrega.cs`

Clase estática **pura** — sin dependencias de Mongo ni HTTP, 100% testeable sin mocks (33 tests en `DetectorCambioEntregaTests.cs`).

| Método | Responsabilidad |
| ------ | ---------------- |
| `LeerEscalar(JToken?)` | Extrae un valor escalar de un JToken, incluso si viene envuelto en Extended JSON (`{"$numberLong": "..."}`) |
| `EsEntregaEnDireccion(pedido, out campoOrigen)` | `TipoEntrega` (o `TipoDeEntrega`, esquema viejo — ~11% de `PedidosInter` lo usa) numéricamente `== 1` |
| `EstaEnOficina(pedido)` | `Estado` (trim + case-insensitive) `== "Reclame en oficina"` |
| `Califica(pedido, out campoOrigen)` | `EstaEnOficina && EsEntregaEnDireccion` — dispara la detección 1 |
| `ConstruirDocumento(...)` | Arma el `BsonDocument` completo (denormalizado) para la detección 1 |
| `ClasificarDesenlace(pedido)` | Aplica la regla de negocio de arriba, retorna `"Entrega"`, `"Devolucion"` o `null` |
| `CalificaResolucion(pedido, out desenlace, out campoOrigen)` | `ClasificarDesenlace != null && EsEntregaEnDireccion` — dispara la detección 2 |
| `ConstruirDocumentoResolucion(pedido, desenlace, ahoraUtc)` | Doc chico: `NumeroGuia`, `EstadoFinal`, `FechaResolucion` |

### Colección `AuditoriaCambioEntregaInter`

| Campo | Tipo | Origen / notas |
| ----- | ---- | --------------- |
| `NumeroGuia` | `string` | `Numeropreenvio` — **índice único** (`ux_numeroGuia`) |
| `PedidoId` | `string` | `_id` del doc en `PedidosInter` |
| `Direccion`, `Ciudad`, `Departamento`, `Destinatario`, `Telefono` | `string` | Denormalizados al momento de la detección 1 |
| `IdTienda`, `Tienda`, `Asesor` | `string` | Denormalizados |
| `TotalRecaudo` | `number` | Cantidad — se suma/promedia en el front, por eso es numérico (no string como el resto de `PedidosInter`) |
| `TipoEntrega` | `number` | Flag `1`/`2`, no un identificador — numérico por la misma razón |
| `FechaCreacionPedido` | `string` | `"Fecha de creación"` del pedido — **anterior** a que Inter lo tocara |
| `Estado` | `string` | Congelado en `"Reclame en oficina"` desde la primera detección — es el registro histórico, no el estado actual |
| `DescripcionAsociada` | `string` | Descripción que Inter reportó junto al evento de oficina |
| `FechaEstadoOficina` | `string` | Fecha/hora que **Inter** reportó para el evento (no la nuestra) |
| `FechaPrimeraDeteccion` | `DateTime` | Cuándo **nuestro sistema** detectó el cambio por primera vez — `$setOnInsert`, nunca cambia |
| `FechaUltimaDeteccion` | `DateTime` | Se refresca cada vez que la guía se vuelve a detectar en oficina |
| `EstadoGestion` | `string` | `"Pendiente"` en la inserción — lo cambia el front (`PUT` vía `/metodoGenerico`) |
| `GestionadoPor` | `object?` | Solo lo escribe el front — auditoría del último cambio de gestión |
| `EstadoFinal` | `string?` | `"Entrega"` \| `"Devolucion"` \| ausente — lo escribe la detección 2 |
| `FechaResolucion` | `DateTime?` | Cuándo se resolvió — junto con `EstadoFinal` |

### Repositorio (`DocumentRepository.cs`)

| Método | Filtro | Modo de escritura |
| ------ | ------ | ------------------ |
| `UpsertCambiosEntregaAsync` | `NumeroGuia` | `$set` (campos que se refrescan) + `$setOnInsert` (`NumeroGuia`, `FechaPrimeraDeteccion`, `EstadoGestion`), `upsert: true` |
| `ActualizarDesenlaceAsync` | `NumeroGuia` | Solo `$set { EstadoFinal, FechaResolucion }`, **`upsert: false`** |

Ambos usan un solo `BulkWriteAsync(..., new BulkWriteOptions { IsOrdered = false })` por corrida — nunca un write por guía.

El índice único `ux_numeroGuia` se crea en el constructor del repositorio, en un `try/catch` vacío (tolera cold starts repetidos sin fallar).

### Backfill único (histórico)

Los documentos de auditoría creados **antes** de esta feature (~1104 en producción) nunca van a recibir `EstadoFinal` de forma orgánica si su pedido en `PedidosInter` ya llegó a un estado terminal (`Pagada`/`Devolución ratificada`), porque esos estados quedan excluidos de `ObtenerPedidosFiltradosAsync` — la lambda ya no los vuelve a tocar.

Se corrió una vez un script de `mongosh` que:

1. Trae los `NumeroGuia` de `AuditoriaCambioEntregaInter` sin `EstadoFinal`.
2. Cruza contra `PedidosInter` por `Numeropreenvio` (`$in`, en chunks de 500).
3. Aplica la misma regla de clasificación en JS.
4. `bulkWrite` de `updateOne` con `$set`, sin upsert — mismo criterio que `ActualizarDesenlaceAsync`.

No quedó como código desplegado — es un script de mantenimiento, se corre una sola vez por ambiente.

---

## Variables de entorno

| Variable | Descripción | Valores |
| -------- | ----------- | ------- |
| `AUDITORIA_CAMBIO_ENTREGA` | Kill switch de **ambas** auditorías (detección + desenlace) — activo salvo que valga explícitamente `"false"` | `"true"` / `"false"` |
| `CADENA_CONEXION` | Cadena de conexión MongoDB (encriptada AES) | String encriptado |
| `DATABASE_NAME` | Nombre de la base de datos MongoDB | `"LogighoDB"` |
| `ID_CLIENTE` | ID cliente crédito Inter (encriptado AES) | String encriptado |
| `USER_AUTH` | Usuario autenticación Inter (encriptado AES) | String encriptado |
| `TOKEN_AUTH` | Token autenticación Inter (encriptado AES) | String encriptado |
| `SUCURSAL_GENERICA` | Sucursal genérica Inter (encriptado AES) | String encriptado |
| `URL_SERVICIO_INTER` | URL base de la API de Interrapidísimo | URL |
| `URL_SERVICIO_AWS` | URL base de servicios internos AWS | URL |

---

## Dependencias externas

| Servicio | Uso |
| -------- | --- |
| `API Interrapidísimo` | `ConsultarEstadosGuiasCliente` — trae el historial de eventos de cada guía |
| `MongoDB — PedidosInter` | Fuente y destino de la actualización principal |
| `MongoDB — ConciliacionPagos` | Cruce para marcar pedidos como `Pagada` (último mes) |
| `MongoDB — TarifasInter` | Cruce para inferir `Trayecto` por valor de flete |
| `MongoDB — AuditoriaCambioEntregaInter` | Colección de esta feature (ver arriba) |
| `/metodoGenerico` (endpoint genérico) | Consumido por el **front**, no por esta lambda — trae `EstadoGuiaLogighoNew` |

---

## Historial de cambios

| Fecha | Autor | Cambio |
| ----- | ----- | ------ |
| 2026-07-31 | Iker Acevedo | Detección 1: registra en `AuditoriaCambioEntregaInter` cuando una guía con `TipoEntrega=1` cambia a "Reclame en oficina". `DetectorCambioEntrega.cs`, índice único `ux_numeroGuia`, kill switch `AUDITORIA_CAMBIO_ENTREGA`. |
| 2026-08-20 | Iker Acevedo | Detección 2: `ClasificarDesenlace`/`CalificaResolucion`/`ConstruirDocumentoResolucion` + `ActualizarDesenlaceAsync` (upsert:false). Reutiliza la regla de clasificación entrega/devolución de `DepurarLiquidacionUseCase.cs`. Agrega `EstadoFinal`/`FechaResolucion` al esquema. `TotalRecaudo`/`TipoEntrega` pasan de string a number en el documento de auditoría (identificadores se mantienen string; cantidades, numéricas). Backfill único vía mongosh para los ~1104 documentos históricos. 33 tests xUnit. |

---

## Observaciones

- La detección 2 corre para **cualquier** pedido que se resuelva, no solo para los que están actualmente en oficina — el filtro real de "¿es una guía auditada?" lo hace Mongo con `upsert:false`, no el código en memoria.
- Si Inter vuelve a mover una guía ya resuelta de nuevo a oficina (caso raro), `EstadoFinal` queda con el valor del último desenlace detectado — no hay lógica de "congelar en el primer desenlace".
- Ambas auditorías están en `try/catch` **independientes** entre sí y del flujo principal — un fallo en la auditoría de desenlace nunca afecta la detección de oficina ni la actualización de `PedidosInter`.
- El consumo de esta colección desde el front es enteramente vía `/metodoGenerico` (ver `ApiLambdaCrudGenericoAOT`) — esta lambda solo escribe, nunca expone un endpoint de lectura propio.
