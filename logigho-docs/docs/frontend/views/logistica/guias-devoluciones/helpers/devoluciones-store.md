## Servicio: DevolucionesStore

**Autor:** Adalberto González
**Fecha:** 2026-07-28
**Estado:** producción
**Ubicación:** `src/app/views/logistica/guias-devoluciones/helpers/devoluciones.store.ts`
**Scope:** `providedIn: 'root'`

---

## ¿Qué hace?

Reemplaza a `FilterService` y al array `datos[]` que antes vivía en el componente. Es la única fuente de verdad del módulo, con estado reactivo via `signal`/`computed` de Angular — ningún consumidor necesita llamar manualmente a `recalcular()`.

---

## Métodos

### `appendLote(rows: DevolucionRow[]): void`

Agrega filas nuevas evitando duplicados por `_id`, usando un `Map<string, true>` interno como índice O(1) — reemplaza el `Set` reconstruido en cada lote de la versión anterior.

### `setAggResult(result: AggWorkerOutput): void`

Publica el resultado del `agregacion.worker` en los signals de chart/tabla.

### `buildFiltrosAgg(): DevolucionesFiltrosAgg`

Empaqueta el estado de filtros/mes en la forma que espera el `agregacion.worker`.

### `toggleOption(key, opt): void`

Agrega/quita una opción de un filtro. `mes` permite máx. 2 selecciones; `ecosistema` sincroniza `tienda`.

### `reset(): void`

Reinicializa todo el estado (usado por el botón "Actualizar").

> El resto de métodos de filtros (`inicializarFiltroMes`, `clearAllFilters`, `removeChip`, etc.) reemplazan uno a uno a los de `FilterService` — misma firma, ahora respaldados por signals en vez de mutación directa de un array.

---

## Endpoints que consume

Ninguno. El acceso HTTP vive en `DevolucionesRepository`.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-28 | Adalberto González | Creación del Store, reemplazando `FilterService` y el array `datos[]` del componente |

---

## Observaciones

- Vive junto al Repository, las Rules y los Web Workers en `helpers/`, mismo patrón que `relacion-despacho`.
