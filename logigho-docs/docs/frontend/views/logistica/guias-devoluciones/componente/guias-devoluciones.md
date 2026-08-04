# Módulo: Guías de Devoluciones

---

## Autor: Adalberto González
Fecha creación: 2026-06-03  
Estado: desarrollo  
Tipo: módulo (1 vista + 2 servicios + 3 workers + 1 modelo + 1 ADR)

---

## Índice

1. [Vista: GuiasDevolucionesComponent](#1-vista-guiasdevolucionescomponent)
2. [Servicio: FilterService](../servicios/filter-service.md)
3. [Servicio: ChartComputerService](../servicios/chart-service.md)
4. [Worker: data-processor.worker](../workers/data-processor-worker.md)
5. [Worker: historico.worker](../workers/historico-worker.md)
6. [Worker-utils: funciones compartidas](../workers/worker-utils.md)
7. [Modelos: Dominio Devoluciones](../modelos/models-guia-devolucion.md)
8. [ADR-001: Arquitectura del módulo](../arquitectura/ADR-guias-devolucion.md)

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
  imports: [CommonModule, FormsModule],
  templateUrl: './guias-devoluciones.component.html',
  styleUrl: './guias-devoluciones.component.scss',
})
export class GuiasDevolucionesComponent implements OnInit, OnDestroy
```

---

### Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `datos` | `DevolucionRow[]` | Array privado con todos los registros acumulados en memoria. Fuente de verdad del componente |
| `historicoListo` | `boolean` | `true` cuando los 2 workers históricos terminaron. Habilita el botón de refresh y cambia el badge de estado |
| `chartData` | `DayPoint[]` | Barras del chart — asignadas desde `ChartComputerService.computar()` |
| `tablaResumen` | `TablaFila[]` | Árbol jerárquico — asignado desde `ChartComputerService.computar()` |
| `tablaDetalle` | `DevolucionRow[]` | Registros del mes activo para la tabla de detalle |
| `tablaDetallePagina` | `DevolucionRow[]` | Subconjunto de `tablaDetalle` para la página actual (50 registros) |
| `skeletonHeights` | `number[]` | Alturas predefinidas para las barras del skeleton — evita que el skeleton sea uniforme |

---

### Servicios y endpoints

| Servicio | Método | Endpoint | Cuándo |
|---|---|---|---|
| `ConsumoGenericoService` | `consultarGenerico()` | `GET metodoGenerico?coleccion=PedidosInter&Estado=...&mcomp=2` | Fases 1 y 2 de carga |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET metodoGenerico?coleccion=Tienda&Estado=ACTIVO&mcomp=1` | Al inicializar, en paralelo con la carga de datos |
| `FilterService` | Todos los métodos | — | Al inicializar y en cada interacción de filtros |
| `ChartComputerService` | `computar()` | — | Cada vez que `datos` o filtros cambian |
| `data-processor.worker` | `postMessage` | — | Fases 1 y 2, para descomprimir payloads |
| `historico.worker` | `postMessage` | — | Fase 3, para cargar el histórico completo |

Ver detalle completo en [FilterService](../servicios/filter-service.md), [ChartComputerService](../servicios/chart-service.md) y los workers ([data-processor](../workers/data-processor-worker.md), [historico](../workers/historico-worker.md)).

---

### Flujo principal

```
ngOnInit()
  → filterService.inicializarFiltroMes()
  → iniciarDotsAnimation()
  → Promise.all([cargarDatos(), cargarTiendas()])

cargarDatos()
  → Fase 1: getPrimeraPagina() x3 estados en paralelo
      → procesarConWorker(payloads)   ← data-processor.worker
      → acumular(nuevos)
      → recalcular()                  ← chart visible para el usuario
      → isLoading = false
  → Fase 2: cargarPaginasRestantes()  ← páginas 2..N en segundo plano
      → procesarConWorker(payloads)
      → acumular() + recalcular()
  → Fase 3: cargarHistoricoConWorker()
      → 2 workers históricos en paralelo
      → por cada lote: acumular() + recalcular()
      → cuando ambos terminan: historicoListo = true

recalcular()
  → filterService.actualizarOpcionesTransp(datos)
  → filterService.actualizarFiltroFechas(datos)
  → chartComputer.computar(datos)     ← aplica filtros + calcula chart y tabla
  → asigna resultado al template
  → actualiza paginación
```

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-06-03 | Adalberto González | Se rediseñó el módulo, pasándolo de vista de Power BI a HTML |

---

### Observaciones

- El componente no contiene lógica de filtros ni de cómputo — todo se delega a `FilterService` y `ChartComputerService`. Su única responsabilidad es orquestar la carga y conectar los servicios con el template.
