---

## Autor: Adalberto González
Fecha creacion: 2026-07-28
Estado: produccion

# Servicio: DevolucionesStore

**Ubicación:** `src/app/views/logistica/guias-devoluciones/helpers/devoluciones.store.ts`
**Scope:** `providedIn: 'root'`

---

## ¿Qué hace?

Única fuente de verdad del módulo. Reemplaza el array `datos[]` del componente y el estado de filtros que antes vivía en `FilterService`. Estado reactivo con `signal`/`computed` — sin necesidad de llamar `recalcular()` manualmente.

---

## Métodos

### `appendLote(rows: DevolucionRow[]): void`

Agrega filas nuevas evitando duplicados por `_id`, usando un `Map` interno como índice O(1).

### `setAggResult(result: AggWorkerOutput): void`

Publica el resultado del `agregacion.worker` en los signals de chart/tabla.

### `buildFiltrosAgg(): DevolucionesFiltrosAgg`

Empaqueta el estado de filtros para enviarlo al `agregacion.worker`.

### `toggleOption(key: FilterKey, opt: string): void`

Agrega o quita una opción de un filtro. `mes` permite máx. 2 selecciones; `ecosistema` sincroniza `tienda`.

> Repetir por cada método. Ver el archivo fuente para el resto (`inicializarFiltroMes`, `clearAllFilters`, `reset`, etc.) — todos siguen el mismo patrón de `signal.update()`.

---

## Endpoints que consume

Ninguno. El acceso HTTP vive en `DevolucionesRepository`.

---

## Historial de cambios

| Fecha      | Autor              | Cambio                                                                          |
| ----------- | -------------------- | -------------------------------------------------------------------------------- |
| 2026-07-28 | Adalberto González | Creación del Store, reemplazando `datos[]` del componente y el estado de `FilterService` |

---

## Observaciones

- Vive junto al Repository, las Rules y los Web Workers en `helpers/`, mismo patrón que `relacion-despacho`.
