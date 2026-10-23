---
autor: Iker Acevedo
fecha_creacion: 2026-10-22
ultima_actualizacion: 2026-10-22
estado: diseño
---

# Colecciones — API pública de guías

Producción usa **Amazon DocumentDB**; PreProd usa **Atlas**. Solo se usan operaciones compatibles con DocumentDB: `findOneAndUpdate` atómico, índices únicos, TTL, sin retryable writes. Los índices se crean en ambos entornos antes del primer uso.

---

## Colecciones nuevas (prefijo `Api`)

| Colección | Contenido | Índices |
| --- | --- | --- |
| `ApiClientes` | Cliente (`ClienteId` = app client de Cognito), nombre, NIT, contacto, estado (`ACTIVO`/`BLOQUEADO`/`SUSPENDIDO_CUPO`), credenciales (M2M o usuario Cognito de `/pedidos`), tiendas con productos y transportadoras permitidas, IPs permitidas, límites. Unifica `ApiUsuariosTiendas` | único `ClienteId` |
| `ApiSolicitudesGuia` | Máquina de estados de cada guía: cliente, tienda, `IdempotencyKey`, hash del body, transportadora, número de guía, montos estimados y reales, errores, key de la traza en S3 | único `{ClienteId, IdempotencyKey}`; `{NumeroGuia}`; `{Estado, FechaActualizacion}` |
| `ApiLotes` / `ApiLoteItems` | Lote, contadores, resultado por guía | `{LoteId, Estado}` |
| `ApiCupoTiendas` | Cupo por tienda (modelo en `cupo.md`) | único `IdTienda` (entero) |
| `ApiMovimientosCupo` | Libro de movimientos de cupo: reserva, consumo, liberación, ajuste | único `{SolicitudId, Tipo}` |
| `ApiContadoresUso` | Guías por cliente y día | único `{ClienteId, Fecha}`, TTL 90 días |
| `ApiAuditoria` | Quién, qué, cuándo, IP, antes y después, `traceId` | `{ClienteId, Fecha}` |

---

## Colecciones existentes (mismo formato que hoy)

| Colección | Uso |
| --- | --- |
| `CargaPedido` | Insert con origen `API`, actualización a `Cargado`/`Rechazado` como hoy |
| `PedidosInter` | Insert con el formato actual (PascalCase, `Estado "Creado"`, `Total Flete/Servicio/Sobreflete`, campos financieros vacíos, `UrlGuia`) |
| `Tienda`, `Productos`, `CiudadesInter`, `TipoEntregaInter`, `CostosTransporteTienda` | Solo lectura, con proyección y caché en memoria de 5–15 min |

**Pendiente de verificar antes de proponer índice único en `PedidosInter.Numeropreenvio`:** duplicados existentes y el tipo mixto (string/número) que ya está documentado.

---

## Fuera de Mongo

Trazas HTTP de transportadoras: S3. Ver [ADR-005](adr-005-trazas-transportadoras.md).
