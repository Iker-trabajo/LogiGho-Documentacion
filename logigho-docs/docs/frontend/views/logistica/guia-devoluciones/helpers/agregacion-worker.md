---

## Autor: Adalberto González
Fecha creacion: 2026-07-28
Estado: produccion

# Servicio: agregacion.worker

**Ubicación:** `src/app/views/logistica/guias-devoluciones/helpers/agregacion.worker.ts`
**Scope:** Web Worker (una sola instancia por ciclo de vida del componente)

---

## ¿Qué hace?

Reemplaza a `ChartComputerService.computar()`. Recibe todas las filas crudas y los filtros activos, y calcula fuera del hilo principal: el filtrado, el chart de barras, la tabla jerárquica y la paginación del detalle.

---

## Métodos

### `onmessage(AggWorkerInput): void`

Único punto de entrada. Aplica filtros, calcula chart y tabla en una sola pasada sobre las filas, y responde con `AggWorkerOutput` vía `postMessage`.

| Parámetro | Tipo             | Descripción                                    |
| ---------- | ----------------- | ------------------------------------------------ |
| `rows`    | `DevolucionRow[]` | Filas crudas en memoria                          |
| `filtros` | `DevolucionesFiltrosAgg` | Filtros activos empaquetados              |
| `exportarDetalle` | `boolean?`  | Si es `true`, incluye `tablaDetalleCompleta` en la salida |

**Retorna:** `AggWorkerOutput` — ver [models-guia-devolucion.md](../modelos/models-guia-devolucion.md).

---

## Endpoints que consume

Ninguno. Recibe datos ya cargados en memoria.

---

## Historial de cambios

| Fecha      | Autor              | Cambio                                                                                          |
| ----------- | -------------------- | -------------------------------------------------------------------------------------------- |
| 2026-07-28 | Adalberto González | Creación, moviendo el cómputo de `ChartComputerService` (hilo principal) a un Web Worker |

---

## Observaciones

- El componente mantiene una bandera `aggPending` para no encolar mensajes redundantes mientras el worker procesa uno anterior.
