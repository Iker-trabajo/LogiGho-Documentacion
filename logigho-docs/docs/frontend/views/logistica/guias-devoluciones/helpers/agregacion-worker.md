## Worker: agregacion.worker

**Autor:** Adalberto González
**Fecha:** 2026-07-28
**Estado:** producción
**Ubicación:** `src/app/views/logistica/guias-devoluciones/helpers/agregacion.worker.ts`
**Tipo:** Web Worker

---

## ¿Qué hace?

Reemplaza a `ChartComputerService.computar()`. Recibe todas las filas crudas y los filtros activos, y calcula fuera del hilo principal: el filtrado, el chart de barras, la tabla jerárquica y la paginación del detalle. El componente lanza una sola instancia al iniciar y la reutiliza durante todo el ciclo de vida.

---

## Mensaje de entrada (`AggWorkerInput`)

| Campo | Tipo | Descripción |
|---|---|---|
| `rows` | `DevolucionRow[]` | Todas las filas crudas en memoria |
| `filtros` | `DevolucionesFiltrosAgg` | Filtros activos ya empaquetados |
| `pagina` / `pageSize` | `number` | Paginación de la tabla de detalle |
| `exportarDetalle` | `boolean?` | Si es `true`, incluye `tablaDetalleCompleta` en la salida (exportar a Excel) |

---

## Mensaje de salida (`AggWorkerOutput`)

Ver detalle completo en [models-guia-devolucion.md](../modelos/models-guia-devolucion.md).

---

## Flujo

```
onmessage(AggWorkerInput)
  → aplicarFiltros(rows, tiendas, filtros)
  → calcularChart(vista, prefixes, ...)
  → calcularTabla(vista, prefixes, ...)
  → tablaDetallePagina = slice según pagina/pageSize
  → si exportarDetalle: agrega tablaDetalleCompleta
  → postMessage(AggWorkerOutput)
```

---

## Endpoints que consume

Ninguno. Recibe los datos ya cargados en memoria.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-28 | Adalberto González | Creación, moviendo el cómputo de `ChartComputerService.computar()` (hilo principal) a un Web Worker dedicado |

---

## Observaciones

- El componente mantiene una bandera `aggPending` para no encolar mensajes redundantes mientras el worker procesa uno anterior (coalescing).
