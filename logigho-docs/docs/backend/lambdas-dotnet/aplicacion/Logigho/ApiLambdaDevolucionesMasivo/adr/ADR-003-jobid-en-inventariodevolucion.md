## Autor:
Fecha creacion: 2026-08-26
Estado: aceptada

# ADR-003 — Agregar JobId a cada fila de InventarioDevolucion

**Autor:** Iker Acevedo
**Fecha:** 2026-08-26
**Estado:** Aceptada

---

## Contexto

Tras [ADR-002](ADR-002-jobdevolucion-solo-contadores.md), el detalle de las guías **resueltas** de un lote vive únicamente en `InventarioDevolucion`, no en el job. El front necesitaba mostrar, para un lote puntual del historial, exactamente qué guías se aceptaron — pero el documento de `InventarioDevolucion` no guardaba ninguna referencia a qué lote (`JobId`) originó cada fila.

La única forma de correlacionarlas era por **ventana de tiempo**: tomar `FechaInicio`/`FechaFin` del job y filtrar `InventarioDevolucion` por fecha dentro de ese rango. Funciona, pero tiene un caso borde real: si el mismo operario (o dos operarios de la misma tienda) corren dos lotes solapados en el tiempo, sus guías se mezclarían en el filtro por fecha.

---

## Opciones consideradas

### Opción A — Agregar `JobId` a cada fila de `InventarioDevolucion`

`RegistrarEnInventarioDevolucionAsync` recibe el `jobId` (que ya conoce `ProcesarJobUseCase` desde el principio) y lo agrega a cada `BsonDocument` que inserta.

**Pros:** correlación exacta, sin ambigüedad posible. El `JobId` no cambia entre continuaciones del Worker (ver [ApiLambdaDevolucionesMasivo.md](../ApiLambdaDevolucionesMasivo.md#qué-pasa-cuando-el-worker-muere-y-se-reactiva--cambia-el-jobid)), así que un lote que corrió 3 veces por continuaciones deja **todas** sus filas con el mismo valor — no hay caso especial que manejar. Costo de implementación mínimo: es un campo más en un `InsertMany` que ya se hacía.
**Contras:** los documentos escritos **antes** de este cambio no tienen `JobId` — sus lotes en el historial muestran la pestaña de "Aceptadas" vacía hasta que ese histórico se pueda recuperar con un backfill (no realizado en esta iteración).

### Opción B — Mantener la correlación por ventana de tiempo, ensanchada con margen

Seguir filtrando por fecha, agregando un margen de 1 minuto por lado para cubrir el desfase entre cuándo el job se marca terminado y cuándo se escribió la última guía.

**Pros:** no requiere ningún cambio de backend ni backfill.
**Contras:** no elimina el problema de fondo — dos lotes solapados de la misma tienda seguirían mezclándose. Es una mitigación parcial, no una solución.

---

## Decisión

**Se eligió:** Opción A.

**Razón:** el costo de agregar un campo a una escritura que ya existía es prácticamente nulo, y elimina el problema de raíz en vez de mitigarlo. El caso de datos históricos sin `JobId` se acepta como una limitación conocida y documentada, no como un blocker — el histórico previo simplemente no muestra el detalle de aceptadas, sin romper nada.

---

## Consecuencias

**Positivas:** `IngresoDevolucionesRepository.obtenerInventarioPorJobId(jobId)` reemplaza la correlación por fecha con un filtro exacto por `JobId`. El front ya no necesita las funciones `rangoFechasFiltro`/`resueltasDelLote` para ese flujo (quedan documentadas como parte del histórico del código, útiles si algún día hace falta correlacionar contra datos previos al cambio).

**Negativas:** los lotes procesados antes de este cambio muestran la pestaña "Aceptadas" vacía en el historial — es una limitación conocida, no un bug. Si se necesitara recuperar ese histórico, haría falta un script de backfill que correlacione por ventana de tiempo (la lógica ya existe en `resueltasDelLote`, solo habría que aplicarla una vez).

---

## Impacto en el código

| Módulo / Repo | Cambio |
| ------------- | ------ |
| `ApiLambdaDevolucionesMasivo` (backend) | `IDevolucionRepository.RegistrarEnInventarioDevolucionAsync` recibe `jobId`; se propaga desde `ProcesarJobUseCase.ProcesarLoteAsync` → `PersistirAsync`; `DevolucionRepository` agrega `{ "JobId", jobId }` al `BsonDocument` |
| `ApiLambdaDevolucionesMasivo.Tests` | Nuevo test `RegistrarEnInventario_GuardaElJobIdEnCadaFila`; firma actualizada en `RepositoriosFalsos.cs` y en los tests de integración existentes |
| `SitioLogiGho` (front) | `IngresoDevolucionesRepository.obtenerInventarioPorJobId(jobId)` reemplaza `obtenerInventarioPorFecha` en `historial-lotes.component.ts` |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-08-26 | Iker Acevedo | Decisión inicial, implementación y documento. |

---

## Referencias

- [ADR-002](ADR-002-jobdevolucion-solo-contadores.md) — decisión que movió el detalle de resueltas a `InventarioDevolucion` y originó la necesidad de esta correlación.
