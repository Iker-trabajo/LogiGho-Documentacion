---

## Autor: Adalberto González
Fecha creacion: 2026-06-03  
Estado: Desarrollo  
Tipo: vista

# Vista: GuiasDevolucionesComponent

**Selector:** `app-guias-devoluciones`  
**Ubicación:** `src/app/views/logistica/guias-devoluciones/guias-devoluciones.component.ts`  
**Acceso:** Autenticado | Rol: Todos

---

## ¿Qué hace?

El módulo de Guías de Devoluciones es un tablero de seguimiento que permite consultar el estado de las guías que han sido enviadas para devolución. Segmentandolas por días en barras individuales, y permintiendo a el usuario mirar el detalle de cuales son esas guias en especifico.

---

## Ruta

| Ruta | Guard | Parámetros de URL |
|---|---|---|
| `/logistica/guias-devoluciones` | `AuthGuard` | — |

---

## Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `datos` | `DevolucionRow[]` | Array privado con todos los registros acumulados en memoria. Fuente de verdad del componente. |
| `historicoListo` | `boolean` | `true` cuando los 2 workers históricos terminaron. Habilita el botón de refresh y cambia el badge de estado. |
| `chartData` | `DayPoint[]` | Barras del chart — asignadas desde `ChartComputerService.computar()` |
| `tablaResumen` | `TablaFila[]` | Árbol jerárquico — asignado desde `ChartComputerService.computar()` |
| `tablaDetalle` | `DevolucionRow[]` | Registros del mes activo para la tabla de detalle |
| `tablaDetallePagina` | `DevolucionRow[]` | Subconjunto de `tablaDetalle` para la página actual (50 registros) |
| `skeletonHeights` | `number[]` | Alturas predefinidas para las barras del skeleton — evita que el skeleton sea uniforme |

---

## Servicios y endpoints

| Servicio | Método | Endpoint | Cuándo |
|---|---|---|---|
| `ConsumoGenericoService` | `consultarGenerico()` | `GET metodoGenerico?coleccion=PedidosInter&Estado=...&mcomp=2` | Fases 1 y 2 de carga |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET metodoGenerico?coleccion=Tienda&Estado=ACTIVO&mcomp=1` | Al inicializar, en paralelo con la carga de datos |
| `FilterService` | Todos los métodos | — | Al inicializar y en cada interacción de filtros |
| `ChartComputerService` | `computar()` | — | Cada vez que `datos` o filtros cambian |
| `data-processor.worker` | `postMessage` | — | Fases 1 y 2, para descomprimir payloads |
| `historico.worker` | `postMessage` | — | Fase 3, para cargar el histórico completo |

---

## Flujo principal

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

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-06-03 | Adalberto González | Se rediseñó el módulo, pasándolo de vista de Power BI a HTML |

---

## Observaciones

- El componente no contiene lógica de filtros ni de cómputo — todo se delega a `FilterService` y `ChartComputerService`. Su única responsabilidad es orquestar la carga y conectar los servicios con el template.
