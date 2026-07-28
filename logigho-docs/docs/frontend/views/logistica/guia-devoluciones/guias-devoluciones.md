---

## Autor: Adalberto González
Fecha creacion: 2026-06-03
Fecha actualizacion: 2026-07-28
Estado: produccion
Tipo: vista

# Vista: GuiasDevolucionesComponent

**Selector:** `app-guias-devoluciones`
**Ubicación:** `src/app/views/logistica/guias-devoluciones/guias-devoluciones.component.ts`
**Acceso:** Autenticado | Rol: Todos

---

## ¿Qué hace?

El módulo de Guías de Devoluciones es un tablero de seguimiento que permite consultar el estado de las guías que han sido enviadas para devolución. Segmenta la información por día en un chart de barras, muestra un árbol jerárquico mes → fecha → tienda → guía, y una tabla de detalle paginada con badges de estado y exportación a Excel independiente por sección.

---

## Ruta

| Ruta | Guard | Parámetros de URL |
|---|---|---|
| `/logistica/guias-devoluciones` | `AuthGuard` | — |

---

## Arquitectura del módulo

El componente es un **orquestador delgado**: no contiene lógica de filtros, cómputo de chart/tabla ni acceso HTTP directo. Todo eso vive en `helpers/`:

| Pieza | Responsabilidad |
|---|---|
| `DevolucionesStore` (signals) | Única fuente de verdad del estado — datos crudos, filtros, resultado del chart/tabla, paginación |
| `DevolucionesRepository` | Todo el acceso HTTP (fases 1/2, tiendas, config del worker histórico) |
| `devoluciones.rules.ts` | Funciones puras sin clase — cálculos de filtros, formateo de badges, nombre corto de tienda |
| `agregacion.worker.ts` | Filtrado + cómputo de chart y tabla, en un Web Worker (nunca en el hilo principal) |
| `historico.worker.ts` / `data-processor.worker.ts` | Descarga y descompresión ZSTD, también en Web Workers |

Este es el mismo patrón que `dashboard-sin-despacho` (Store + Repository + Workers), adaptado con `devoluciones.rules.ts` en lugar de un service de Angular para las funciones puras — así el módulo no depende de `@Injectable` para lógica que no necesita inyección de dependencias.

---

## Decoradores y configuración técnica

```typescript
@Component({
    selector: 'app-guias-devoluciones',
    standalone: true,
    imports: [CommonModule, FormsModule, TourGuiadoComponent],
    templateUrl: './guias-devoluciones.component.html',
    styleUrls: ['./guias-devoluciones.component.scss'],
})
export class GuiasDevolucionesComponent implements OnInit, OnDestroy
```

---

## Propiedades clave

Casi todas son getters que leen directamente del `DevolucionesStore` (signals) — el componente no mantiene su propio estado de datos.

| Propiedad | Tipo | Descripción |
|---|---|---|
| `isLoading` | `boolean` | `store.loading()` — activa skeleton del chart y deshabilita los 3 botones de exportar |
| `historicoListo` | `boolean` | `store.historicoListo()` — `true` cuando los 2 workers históricos terminaron |
| `chartData` | `DayPoint[]` | `store.chartData()` — barras publicadas por `agregacion.worker` |
| `tablaResumen` | `TablaFila[]` | `store.tablaResumen()` — árbol jerárquico mes → fecha → tienda → guía |
| `tablaDetallePagina` | `DevolucionRow[]` | `store.tablaDetallePagina()` — página actual (50 filas) de la tabla de detalle |
| `tablaDetalle` | `{ length: number }` | Wrapper sobre `store.totalDetalle()` para no romper el template que usa `.length` |
| `filters` | `FilterState[]` | `store.filters()` — estado de los 7 filtros (antes vivía en `FilterService`, ahora en el Store) |
| `hayFiltrosActivos` | `boolean` | `store.hayFiltrosActivos()` |
| `skeletonHeights` | `number[]` | Alturas predefinidas para las barras del skeleton — evita que el skeleton sea uniforme |
| `tourAbierto` / `tourSteps` | `boolean` / `TourStep[]` | Estado y pasos del tour guiado (`TourGuiadoComponent`) |

---

## Servicios y endpoints

| Servicio | Método | Cuándo |
|---|---|---|
| `DevolucionesRepository` | `getPrimeraPagina()`, `getPagina()` | Fases 1 y 2 de carga |
| `DevolucionesRepository` | `getTiendas()` | Al inicializar, en paralelo con la carga de datos |
| `DevolucionesRepository` | `buildHistoricoWorkerConfig()` | Antes de lanzar los 2 workers de la fase 3 |
| `DevolucionesStore` | Todos los métodos | Al inicializar y en cada interacción de filtros/paginación |
| `data-processor.worker` | `postMessage` | Fases 1 y 2, para descomprimir payloads |
| `historico.worker` | `postMessage` | Fase 3, para cargar el histórico completo (últimos 4 meses) |
| `agregacion.worker` | `postMessage` | Cada vez que cambian `rawRows`, filtros o página — calcula chart y tabla |

---

## Flujo principal

```
ngOnInit()
  → store.inicializarFiltroMes()      ← últimos 4 meses, no 8
  → iniciarDotsAnimation()
  → iniciarAggWorker()                ← lanza el Web Worker de agregación una sola vez
  → Promise.all([cargarDatos(), cargarTiendas()])

cargarDatos()
  → Fase 1: repo.getPrimeraPagina() x7 estados en paralelo (ventana: últimos 60 días)
      → procesarConWorker(payloads)   ← data-processor.worker
      → store.appendLote(nuevos)      ← dedup O(1) via Map interno
      → dispararAggregation()         ← chart visible para el usuario
      → store.setLoading(false)
  → Fase 2: cargarPaginasRestantes()  ← páginas 2..N en segundo plano
      → store.appendLote() + dispararAggregation()
  → Fase 3: cargarHistoricoConWorker()
      → 2 workers históricos en paralelo (ventana: últimos 4 meses)
      → por cada lote: store.appendLote() + dispararAggregation()
      → cuando ambos terminan: store.setHistoricoListo(true)

dispararAggregation()
  → aggWorker.postMessage({ rows: store.rawRows(), tiendas: store.tiendas(),
                             filtros: store.buildFiltrosAgg(), pagina, pageSize })
  → coalescing: si ya hay una agregación en curso, marca aggPending y la
    vuelve a disparar cuando la actual termine (nunca encola mensajes)

agregacion.worker → onmessage
  → store.setAggResult(data)          ← publica chart/tabla al Store
  → store.actualizarOpcionesTransp()
  → store.actualizarFiltroFechas()
```

---

## Exportar a Excel (por panel, no global)

Cada uno de los 3 paneles visuales tiene su propio botón "Exportar Excel", exportando **solo lo que se ve en pantalla en ese momento** (respetando los filtros activos):

| Método | Exporta |
|---|---|
| `exportarChartExcel()` | El detalle diario (recibida/pendiente/total) del chart de barras |
| `exportarResumenExcel()` | El árbol mes/fecha ya filtrado, tal como se ve en la tabla resumen |
| `exportarDetalleExcel()` | **Todo** el detalle filtrado (todas las páginas, no solo las 50 filas visibles) — se lo pide al `agregacion.worker` con `exportarDetalle: true` para no recorrer el dataset completo en el hilo principal |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-06-03 | Adalberto González | Se rediseñó el módulo, pasándolo de vista de Power BI a HTML |
| 2026-07-28 | Adalberto González | Histórico reducido de 8 a 4 meses; agregado `&campos=` en las consultas a `PedidosInter` (reduce ~951 KB a ~317 KB por página); corregido bug de header `headersecurity`/`headerSecurity` que dejaba el header de seguridad `undefined` en las requests del histórico |
| 2026-07-28 | Adalberto González | Exportación a Excel dividida en 3 botones independientes (chart, resumen, detalle completo), en vez de un único botón global |

---

## Observaciones

- El componente no contiene lógica de filtros ni de cómputo — todo se delega al `DevolucionesStore`, a `devoluciones.rules.ts` y al `agregacion.worker`. Su única responsabilidad es orquestar la carga y conectar el Store con el template.
- `estadoBadge(estado)` y `tiendaCorta(nombre)` son wrappers delgados sobre funciones puras de `devoluciones.rules.ts`, expuestos como métodos del componente para poder llamarlos desde el template.
- El tour guiado se oculta en pantallas ≤860px (`button.btn-tour { display: none }`) — depende de posicionamiento por hover que no tiene sentido en móvil.
