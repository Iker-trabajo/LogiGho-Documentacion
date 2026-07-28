# Módulo: Guías de Devoluciones

---

## Autor: Adalberto González
Fecha creación: 2026-06-03
Fecha actualización: 2026-07-28
Estado: produccion
Tipo: módulo (1 vista + Store + Repository + Rules + 1 workers + 1 modelo + ADR)

---

## Índice

1. [Vista: GuiasDevolucionesComponent](#1-vista-guiasdevolucionescomponent)
2. [Servicio: DevolucionesStore](helpers/devoluciones-store.md)
3. [Servicio: DevolucionesRepository](helpers/devoluciones-repository.md)
4. [Servicio: devoluciones.rules](helpers/devoluciones-rules.md)
5. [Worker: agregacion.worker](helpers/agregacion-worker.md)
6. [Modelos: Dominio Devoluciones](modelos/models-guia-devolucion.md)
7. [ADR: Migración a Store + Repository + Workers](arquitectura/ADR-002-guias-devolucion-store.md)

---

## 1. Vista: GuiasDevolucionesComponent

**Selector:** `app-guias-devoluciones`  
**Ubicación:** `src/app/views/logistica/guias-devoluciones/guias-devoluciones.component.ts`  
**Acceso:** Autenticado | Rol: Todos

---

### ¿Qué hace? (para el usuario)

Es la pantalla de seguimiento del estado de las guías enviadas para devolución. Al abrirla, carga automáticamente los registros de devoluciones y muestra:

- **Un chart de barras apiladas** segmentado por día, diferenciando guías recibidas (completadas) y pendientes (ratificadas).
- **Una tabla resumen jerárquica** (árbol expandible: mes → fecha → tienda → guía) con los totales acumulados de cada nivel.
- **Una tabla de detalle** paginada con los registros del mes activo, incluyendo el valor declarado total.
- **Filtros** por mes, fecha, tipo de día, tienda, ecosistema, transportadora y estado, cada uno como dropdown de selección múltiple con chips.
- Un badge de estado que indica cuándo terminó de cargar el histórico completo y habilita el botón de refresh.

---

### Ruta

```
logistica/guias-devoluciones
```

| Ruta | Guard | Parámetros de URL |
|---|---|---|
| `/logistica/guias-devoluciones` | `AuthGuard` | — |

---

### Decoradores y configuración técnica

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

### Propiedades clave

> Casi todas son getters que leen directamente del `DevolucionesStore` (signals) — el componente no mantiene su propio array de datos.

| Propiedad | Tipo | Descripción |
|---|---|---|
| `isLoading` | `boolean` | `store.loading()` — activa skeleton del chart y deshabilita los botones de exportar |
| `historicoListo` | `boolean` | `store.historicoListo()` — `true` cuando los 2 workers históricos terminaron |
| `chartData` | `DayPoint[]` | `store.chartData()` — barras publicadas por `agregacion.worker` |
| `tablaResumen` | `TablaFila[]` | `store.tablaResumen()` — árbol jerárquico mes → fecha → tienda → guía |
| `tablaDetallePagina` | `DevolucionRow[]` | `store.tablaDetallePagina()` — página actual (50 filas) de la tabla de detalle |
| `filters` | `FilterState[]` | `store.filters()` — estado de los 7 filtros, ahora en el Store |
| `skeletonHeights` | `number[]` | Alturas predefinidas para las barras del skeleton — evita que el skeleton sea uniforme |

---

### Servicios y endpoints

| Servicio | Método | Endpoint | Cuándo |
|---|---|---|---|
| `DevolucionesRepository` | `getPrimeraPagina()`, `getPagina()` | `GET metodoGenerico?coleccion=PedidosInter&...` | Fases 1 y 2 de carga |
| `DevolucionesRepository` | `getTiendas()` | `GET metodoGenerico?coleccion=Tienda&Estado=ACTIVO&mcomp=1` | Al inicializar, en paralelo con la carga de datos |
| `DevolucionesStore` | Todos los métodos | — | Al inicializar y en cada interacción de filtros/paginación |
| `agregacion.worker` | `postMessage` | — | Cada vez que cambian datos, filtros o página — calcula chart y tabla |
| `historico.worker` | `postMessage` | — | Fase 3, para cargar el histórico (últimos 4 meses) |

Ver detalle completo en [DevolucionesStore](helpers/devoluciones-store.md), [DevolucionesRepository](helpers/devoluciones-repository.md), [devoluciones.rules](helpers/devoluciones-rules.md) y [agregacion.worker](helpers/agregacion-worker.md).

---

### Flujo principal

```
ngOnInit()
  → store.inicializarFiltroMes()      ← últimos 4 meses, no 8
  → iniciarDotsAnimation()
  → iniciarAggWorker()                ← lanza el Web Worker de agregación una sola vez
  → Promise.all([cargarDatos(), cargarTiendas()])

cargarDatos()
  → Fase 1: repo.getPrimeraPagina() x7 estados en paralelo (últimos 60 días)
      → procesarConWorker(payloads)   ← data-processor.worker
      → store.appendLote(nuevos)      ← dedup O(1) via Map interno
      → dispararAggregation()         ← chart visible para el usuario
      → store.setLoading(false)
  → Fase 2: cargarPaginasRestantes()  ← páginas 2..N en segundo plano
      → store.appendLote() + dispararAggregation()
  → Fase 3: cargarHistoricoConWorker()
      → 2 workers históricos en paralelo (últimos 4 meses)
      → por cada lote: store.appendLote() + dispararAggregation()
      → cuando ambos terminan: store.setHistoricoListo(true)

dispararAggregation()
  → aggWorker.postMessage({ rows, tiendas, filtros, pagina, pageSize })
  → coalescing: si ya hay una agregación en curso, se re-dispara al terminar

agregacion.worker → onmessage
  → store.setAggResult(data)          ← publica chart/tabla al Store
  → store.actualizarOpcionesTransp()
  → store.actualizarFiltroFechas()
```

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-06-03 | Adalberto González | Se rediseñó el módulo, pasándolo de vista de Power BI a HTML |
| 2026-07-28 | Adalberto González | Migrado a Store + Repository + Rules + Workers (`helpers/`), eliminando `FilterService` y `ChartComputerService`; histórico reducido de 8 a 4 meses; exportación a Excel dividida por panel; rediseño visual con badges de estado y tienda acortada en la tabla de detalle |

---

### Observaciones

- El componente no contiene lógica de filtros ni de cómputo — todo se delega a `DevolucionesStore`, `devoluciones.rules.ts` y `agregacion.worker`. Su única responsabilidad es orquestar la carga y conectar el Store con el template.
- `estadoBadge()` y `tiendaCorta()` son wrappers delgados sobre funciones puras de `devoluciones.rules.ts`, expuestos como métodos del componente para poder llamarlos desde el template.
