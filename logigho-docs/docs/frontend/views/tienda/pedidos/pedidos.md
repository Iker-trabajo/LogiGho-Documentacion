---

## Autor: Adalberto González

Fecha creacion: 2026-06-05  
Estado: Desarrollo 
Tipo: vista

# Vista: PedidosComponent

**Selector:** `app-pedidos`  
**Ubicación:** `src/app/views/tienda/pedidos`  
**Acceso:** Autenticado | Roles: `Administrador`, `Tienda`, `Controller`, `CEO`, `Dropshipper`, `COO`, `Logistica`, `Accionista`, `GLOBAL ACCOUNT`, `TiendaNW`, `BPO`

---

## ¿Qué hace?

Este módulo funciona como el centro de control para la gestión de pedidos de envío de una tienda o multiples tiendas.
Desde una única pantalla, los usuarios pueden consultar, buscar, filtrar, importar y exportar órdenes gestionadas por las transportadoras

---

## Ruta

| Ruta | Guard | Parámetros de URL |
| --- | --- | --- |
| `/app/tienda/pedidos` | `AuthGuard` | — |

---

## Decoradores

| Decorador | Nombre | Tipo | Descripción |
| --- | --- | --- | --- |
| `@Input` | `searchValue` | `string` | Búsqueda genérica sobre las filas visibles |
| `@Input` | `searchValuePedidos` | `string` | Búsqueda externa que filtra sobre `rowsMemoryDate` |

---

## Propiedades clave

| Propiedad | Tipo | Descripción |
| --- | --- | --- |
| `rowsMemoryDate` | `TablaRow[]` | Copia completa de los datos cargados. Es la fuente de verdad para filtros y búsqueda en memoria. No se modifica al filtrar |
| `filteredRows` | `TablaRow[]` | Filas que se pasan a `TablesComponent` tras aplicar filtros de estado |
| `statusCategories` | `object` | Mapa de los 3 estados principales y sus sub-estados. Cada sub-estado contiene los valores exactos del campo `Estado` en MongoDB |
| `filtroEstado` | `string[]` | Array de valores de estado activos. Si está vacío, se muestran todos los pedidos |
| `totalRegistros` | `number` | Total real de registros en la colección MongoDB, no la cantidad cargada en memoria |
| `paginationCache` | `PaginationCacheService` | Caché IndexedDB ligado a la sesión del usuario. Se invalida automáticamente al cerrar sesión |

---

## Servicios y endpoints

| Servicio | Método | Endpoint | Cuándo |
| --- | --- | --- | --- |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET metodoGenerico?coleccion=PedidosInter` | Carga inicial y búsquedas con filtros |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET metodoGenerico?coleccion=Pedido` | Cuando la integración activa es Hoko |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET metodoGenerico?coleccion=Tienda` | Al inicializar para poblar el dropdown de tiendas |
| `PaginationCacheService` | `getCachedProcessed()` | IndexedDB local | Antes de cada carga inicial para evitar peticiones redundantes |
| `PaginationCacheService` | `setCachedProcessed()` | IndexedDB local | Tras procesar los datos de la carga inicial |

---

## Flujo principal

La carga de datos usa un Web Worker (`data-processor.worker.ts`) para descomprimir ZSTD en un hilo separado sin bloquear la UI.

```
ngOnInit()
  -> fetchTableDataTienda()       — puebla dropdown de tiendas
  -> fetchTableData()
        -> [paralelo] respControl (página 1, lote mínimo) + getCachedProcessed()
              -> si hay caché válido de sesión → processData(caché)   [carga rápida]
              -> si no hay caché:
                    -> resp2 (página 2, 8000 registros)
                    -> processWithWorker([respControl, resp2])         [Web Worker]
                    -> setCachedProcessed()
                    -> processData()

processData()
  -> renombra Numeropreenvio → Guia
  -> formatea monedas (COP)
  -> genera columnas
  -> puebla rowsMemoryDate y rows
  -> extrae opciones de filtros (productos, estados, transportadoras)
  -> applyFilter()
```

---

## Historial de cambios

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-06-05 | Adalberto González | Se ajusto la carga inicial optimizada con 2 peticiones en paralelo. Caché IndexedDB por sesión via `PaginationCacheService`. |

---

## Observaciones