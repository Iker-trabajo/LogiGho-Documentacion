## Autor:
Fecha creacion: 2026-08-26
Estado: aceptada

# ADR-002 — JobDevolucion solo guarda contadores y el detalle de rechazadas, no de todas las guías

**Autor:** Iker Acevedo
**Fecha:** 2026-08-24
**Estado:** Aceptada

---

## Contexto

La primera versión de `JobDevolucion` guardaba un array `Resultados: List<GuiaProcesada>` con el detalle completo de **cada** guía procesada, resuelta o rechazada. Con archivos de hasta 3.000 guías, ese array podía crecer sin control dentro de un solo documento de MongoDB/DocumentDB — el mismo tipo de problema de fondo (documentos/consultas sin límite) que ya había causado los picos de CPU del módulo legacy que este proyecto reemplaza.

---

## Opciones consideradas

### Opción A — Guardar solo contadores, y el detalle de rechazadas nada más

Reemplazar `Resultados` por 3 contadores (`TotalResueltas`, `TotalYaProcesadas`, `TotalRechazadas`) y un array `Rechazadas: List<GuiaProcesada>` — el detalle de las **resueltas** vive aparte, en `InventarioDevolucion` (que de todas formas se escribe siempre, por cada guía procesada).

**Pros:** el documento del job crece de forma acotada — en el peor caso (100% de rechazo), el array `Rechazadas` tiene como máximo `MaximoGuiasPorJob` (3.000) elementos, pero en el caso típico (la mayoría se resuelve bien) es mucho más chico. El front necesita el detalle de rechazadas para explicarle al operario qué falló y por qué — ese es justo el array que se conserva.
**Contras:** para ver el detalle de las guías **resueltas**, el front tiene que consultar `InventarioDevolucion` aparte, con una query distinta.

### Opción B — Mantener el detalle completo, pero paginado en sub-documentos

Partir `Resultados` en sub-documentos de tamaño fijo (por ejemplo, 500 guías cada uno), referenciados desde el job principal.

**Pros:** no pierde ningún detalle en el documento del job.
**Contras:** mucho más complejo de leer, escribir y mantener consistente. No resuelve el problema de fondo: seguiría siendo información duplicada, porque `InventarioDevolucion` ya guarda el detalle completo de cada guía procesada, resuelta o rechazada.

---

## Decisión

**Se eligió:** Opción A.

**Razón:** el detalle de las guías resueltas ya vive en `InventarioDevolucion` — guardarlo también en el job sería duplicar el dato sin ninguna ganancia, y con el riesgo real de que el documento creciera sin techo en archivos grandes. El detalle de rechazadas sí se conserva en el job porque es lo que el operario necesita ver **inmediatamente** al terminar un lote, sin depender de una segunda colección, y las rechazadas típicamente son una fracción pequeña del total.

---

## Consecuencias

**Positivas:** el tamaño del documento `JobDevolucion` queda acotado de forma predecible. `ConsultarJobUseCase` sigue siendo una consulta barata (`findOne`) sin importar cuántas guías tenga el job.

**Negativas:** ver el detalle de las guías **aceptadas** de un lote requiere una segunda consulta, contra `InventarioDevolucion` — inicialmente correlacionada por ventana de tiempo (imprecisa con lotes solapados), corregida después agregando `JobId` a cada fila (ver [ADR-003](ADR-003-jobid-en-inventariodevolucion.md)).

Esta decisión expuso además un bug real de idempotencia: `ObtenerGuiasYaProcesadasAsync` originalmente marcaba como "ya procesada" cualquier guía con una fila en `InventarioDevolucion`, sin importar si esa fila era una rechazada o una resuelta — bloqueando permanentemente el reintento de una guía que falló por un motivo corregible (como `SinPermisoTienda`, después de asignarle la tienda al usuario). Se corrigió filtrando por `Validacion=="OK"`.

---

## Impacto en el código

| Componente | Cambio |
| ---------- | ------ |
| `JobDevolucion.cs` | `Resultados` reemplazado por `TotalResueltas`/`TotalYaProcesadas`/`TotalRechazadas`/`Rechazadas` |
| `JobRepository.ActualizarProgresoAsync` | `PushEach` solo sobre `Rechazadas`, `Inc` sobre los 3 contadores |
| `ConsultarJobUseCase.Mapear` | Lee los contadores directo, `Detalle` devuelve `job.Rechazadas` |
| `DevolucionRepository.ObtenerGuiasYaProcesadasAsync` | Filtro `Validacion=="OK"` agregado (fix del bug de idempotencia) |
| Frontend `LoteHistorial`, `EstadoLoteResponse` | Modelos alineados a contadores + `detalleRechazadas` |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-08-24 | Iker Acevedo | Decisión inicial, rediseño y fix del bug de idempotencia expuesto por el cambio. |

---

## Referencias

- [ADR-003](ADR-003-jobid-en-inventariodevolucion.md) — cómo se recuperó la correlación exacta con el detalle de resueltas que este ADR movió a otra colección.
