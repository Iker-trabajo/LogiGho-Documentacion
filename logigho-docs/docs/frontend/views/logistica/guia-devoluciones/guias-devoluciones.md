# Módulo: Guías de Devoluciones

---

## Autor: Adalberto González
Fecha creación: 2026-06-03  
Estado: produccion  
Tipo: módulo (1 vista + Store + Repository + Rules + 3 Workers)

---

## Índice

1. [Vista: GuiasDevolucionesComponent](componente/guias-devoluciones.md)
2. [Servicio: DevolucionesStore](helpers/devoluciones-store.md)
3. [Servicio: DevolucionesRepository](helpers/devoluciones-repository.md)
4. [Servicio: devoluciones.rules](helpers/devoluciones-rules.md)
5. [Servicio: agregacion.worker](helpers/agregacion-worker.md)
6. [Servicio: historico.worker](helpers/historico-worker.md)
7. [Servicio: data-processor.worker](helpers/data-processor-worker.md)
8. [Servicio: worker-utils](helpers/worker-utils.md)
9. [Dominio: Devoluciones (modelos)](modelos/models-guia-devolucion.md)
10. [ADR-001: Arquitectura por capas (reemplazada)](arquitectura/ADR-guias-devolucion.md)
11. [ADR-002: Migración a Store + Repository + Workers](arquitectura/ADR-002-guias-devolucion-store.md)

---

## 1. Vista: GuiasDevolucionesComponent

**Selector:** `app-guias-devoluciones`  
**Ubicación:** `src/app/views/logistica/guias-devoluciones/guias-devoluciones.component.ts`  
**Acceso:** Logística → Guías de Devoluciones

---

### ¿Qué hace? (para el usuario)

Es el tablero de seguimiento de guías enviadas a devolución. Al abrirla, carga automáticamente las devoluciones en 3 fases y muestra:

- **Un chart de barras** con las devoluciones recibidas y pendientes por día.
- **Filtros** por mes, fecha, tipo de día, tienda, ecosistema, transportadora y estado.
- **Una tabla resumen** en árbol jerárquico: mes → fecha → tienda → guía.
- **Una tabla de detalle** paginada, con badges de estado y tienda acortada.
- Cada sección tiene su propio botón para exportar a Excel, respetando los filtros activos.
- Puede actualizar los datos con un botón dedicado en el topbar.
- Tiene un tour guiado que explica cada sección (oculto en móvil).

---

### Ruta

```
logistica/guias-devoluciones
```

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

### Arquitectura del módulo

El componente es un **orquestador delgado** — no contiene lógica de filtros, cómputo de chart/tabla ni acceso HTTP directo. Todo vive en `helpers/`:

| Pieza | Responsabilidad |
|---|---|
| `DevolucionesStore` | Única fuente de verdad — datos crudos, filtros, resultado de chart/tabla, paginación |
| `DevolucionesRepository` | Todo el acceso HTTP |
| `devoluciones.rules.ts` | Funciones puras sin clase — cálculos de filtros, badges, tienda corta |
| `agregacion.worker.ts` | Filtrado + cómputo de chart y tabla, en Web Worker |
| `historico.worker.ts` / `data-processor.worker.ts` | Descarga y descompresión ZSTD |

Mismo patrón que `dashboard-sin-despacho` y `relacion-despacho`.

---

### Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `isLoading` | `boolean` | Activa el skeleton del chart y deshabilita los botones de exportar |
| `historicoListo` | `boolean` | `true` cuando los 2 workers históricos terminaron |
| `chartData` | `DayPoint[]` | Barras del chart, publicadas por `agregacion.worker` |
| `tablaResumen` | `TablaFila[]` | Árbol jerárquico mes → fecha → tienda → guía |
| `tablaDetallePagina` | `DevolucionRow[]` | Página actual (50 filas) de la tabla de detalle |
| `filters` | `FilterState[]` | Estado de los 7 filtros multi-select |

---

### Servicios y endpoints

| Servicio | Uso |
|---|---|
| `DevolucionesRepository` | Fases 1/2/3 de carga, tiendas, config del worker histórico |
| `DevolucionesStore` | Estado reactivo de todo el módulo |
| `agregacion.worker` | Cómputo de chart/tabla fuera del hilo principal |

**Patrón de URL:**
```
metodoGenerico?coleccion=PedidosInter&Tienda=<TIENDAS>&Estado=<ESTADO>&fechasFiltro=<RANGO>&campos=<CAMPOS>&mcomp=2
```

**Endpoints por colección:**

| Colección | Propósito |
|---|---|
| `PedidosInter` | Devoluciones (fases 1/2/3) |
| `Tienda` | Ecosistemas para el filtro de tienda |

---

### Flujo principal

```
ngOnInit()
  └─► store.inicializarFiltroMes() + iniciarAggWorker()
      └─► Promise.all([cargarDatos(), cargarTiendas()])

cargarDatos()
  ├─► Fase 1: 7 estados en paralelo (últimos 60 días) → chart visible rápido
  ├─► Fase 2: páginas restantes en segundo plano
  └─► Fase 3: histórico completo (últimos 4 meses) vía 2 workers

Usuario interactúa con filtros
  └─► store.toggleOption() → dispararAggregation() → agregacion.worker recalcula

Usuario exporta un panel
  └─► exportarChartExcel() / exportarResumenExcel() / exportarDetalleExcel()
```

---

### Historial de cambios

| Fecha | Cambio |
|---|---|
| 2026-06-03 | Creación del módulo, migrado de Power BI a HTML |
| 2026-07-28 | Migración a Store + Repository + Rules + Workers; histórico reducido a 4 meses; exportación a Excel por panel; rediseño visual y badges de estado |

---

### Observaciones

- El componente no contiene lógica de filtros ni de cómputo — todo se delega a `DevolucionesStore`, `devoluciones.rules.ts` y `agregacion.worker`.
- El tour guiado se oculta en pantallas ≤860px.
