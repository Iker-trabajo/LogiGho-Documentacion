---

## Autor:
Fecha creacion: 2026-08-20
Estado: aceptada

# ADR-001 — Persistir el desenlace final en la propia colección de auditoría

**Autor:** Iker Acevedo
**Fecha:** 2026-08-20
**Estado:** Aceptada

---

## Contexto

`AuditoriaCambioEntregaInter` registra cuándo una guía cambia a "Reclame en oficina", pero el campo `Estado` del documento queda **congelado** ahí para siempre — cuando el pedido finalmente se entrega o se devuelve, el doc de auditoría no se entera. El PO necesitaba poder ver, de todas las guías que pasaron por oficina, cuántas terminaron en entrega vs devolución vs siguen sin resolver — y una gráfica que lo muestre.

Restricción técnica clave: el endpoint genérico `/metodoGenerico` (`ApiLambdaCrudGenericoAOT`) **prohíbe `$lookup`** en sus pipelines de agregación (`ValidadorPipeline.cs`) — no se puede cruzar `AuditoriaCambioEntregaInter` con `PedidosInter` del lado de Mongo.

---

## Opciones consideradas

### Opción A — Persistir el desenlace en la propia auditoría

La lambda `ApiLambdaProcesoAutomaticoInter`, que ya resuelve el `Estado` final de cada pedido en cada corrida, clasifica ese estado (Entrega/Devolución) y lo suma al mismo `BulkWrite` que ya hace. Se agrega `EstadoFinal`/`FechaResolucion` al documento.

**Pros:** costo extra ≈ 0 (mismo bulk write, sin round-trips nuevos). El front ya carga toda la colección en memoria (paginación 100% client-side), así que la gráfica se calcula ahí mismo sin pipeline nuevo. Consistente con el patrón ya usado por `DepurarLiquidacionUseCase.cs` para la misma pregunta de negocio.
**Contras:** duplica un dato (el desenlace vive en 2 colecciones si se cuenta `PedidosInter`). Requiere backfill único para los documentos históricos que ya salieron del loop activo de la lambda.

### Opción B — Proyección en consulta (cruzar con `PedidosInter` en cada carga de la vista)

No tocar la lambda. Cada carga de la vista: traer `AuditoriaCambioEntregaInter` + traer `PedidosInter` filtrado por `Numeropreenvio in [...]` (usando el soporte de `$in` automático por comas de `/metodoGenerico`) + cruzar en el cliente.

**Pros:** no duplica dato, no toca `DetectorCambioEntrega.cs` ni el flujo de la lambda.
**Contras:** 2 llamadas HTTP en cada apertura de la vista, en vez de una. La gráfica de conteo por desenlace **no se puede agregar en Mongo** sin `$lookup` — habría que sumarla a mano en el cliente igual, así que no gana nada ahí. Más lento en la práctica.

---

## Decisión

**Se eligió:** Opción A.

**Razón:** el costo de escritura es prácticamente nulo (se aprovecha un bulk write que ya existe), y resuelve de una vez tanto la tabla como la gráfica sin pipeline nuevo. La Opción B no elimina el problema de fondo (no hay `$lookup`) — solo lo mueve del backend al frontend, y encima agrega latencia en cada apertura de la vista.

---

## Consecuencias

**Positivas:** `EstadoFinal` está disponible de inmediato en el mismo documento que ya usa la tabla y el modal de gráficas — cero llamadas adicionales, cero lógica de cruce en el cliente.
**Negativas:** los documentos creados antes de esta feature (~1104 en producción) necesitaron un backfill único vía `mongosh` — ver la sección correspondiente en el doc de la lambda. Si `PedidosInter` y `AuditoriaCambioEntregaInter` alguna vez se desincronizan, `EstadoFinal` puede quedar desactualizado hasta que la guía vuelva a pasar por el loop de la lambda.

---

## Impacto en el código

| Módulo / Repo | Cambio |
| ------------- | ------ |
| `LambdasLogiGho` | `DetectorCambioEntrega.ClasificarDesenlace`/`CalificaResolucion`/`ConstruirDocumentoResolucion`, `DocumentRepository.ActualizarDesenlaceAsync` (upsert:false) |
| `SitioLogiGho` | `EstadoFinal`/`FechaResolucion` en `CambioEntregaInter`, badge en tabla, timeline del modal de detalle, 3 gráficas en `cambio-entrega-analisis` |

Decisión relacionada, dentro del mismo cambio: `TotalRecaudo` y `TipoEntrega` pasan de `string` a `number` en el documento — pero **no** los campos identificadores (`NumeroGuia`, `Telefono`, `IdTienda`), que se mantienen `string` a propósito. Regla aplicada: un identificador nunca se opera aritméticamente (nunca se suma un teléfono), así que aunque "parezca número" se guarda como string — evita 2 problemas reales: JS/BSON pierden precisión en enteros grandes (`NumeroGuia` de 12 dígitos está cerca del límite de precisión segura de un `double`), y un teléfono de 10 dígitos ya supera el rango de `Int32` de Mongo. `TotalRecaudo` sí es una cantidad que se suma/promedia en el front, así que se beneficia de ser numérico — y como el peso colombiano no maneja centavos, no hay riesgo de los problemas de precisión de punto flotante que sí aplicarían a una moneda con decimales.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-08-20 | Iker Acevedo | Decisión inicial y documento. |

---

## Referencias

- [`ApiLambdaProcesoAutomaticoInter`](../../../../../backend/lambdas-dotnet/aplicacion/interrapidismo/ApiLambaProcesoAutomaticoInter/ApiLambaProcesoAutomaticoInter.md) — lambda que escribe `EstadoFinal`.
- `DepurarLiquidacionUseCase.cs` (`ApiLambdaLiquidacionesLogighoAOT`) — fuente original de la regla de clasificación Entrega/Devolución.
