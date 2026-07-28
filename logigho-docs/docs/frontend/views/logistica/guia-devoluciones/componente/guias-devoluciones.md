---

## Autor: Adalberto González

Fecha creacion: 2026-06-03
Estado: produccion
Tipo: vista

# Vista: GuiasDevolucionesComponent

**Selector:** `app-guias-devoluciones`
**Ubicación:** `src/app/views/logistica/guias-devoluciones/guias-devoluciones.component.ts`
**Acceso:** Autenticado | Rol: Todos

---

## ¿Qué hace?

Tablero de seguimiento de guías enviadas a devolución. Muestra un chart de barras por fecha, un árbol jerárquico mes → fecha → tienda → guía, y una tabla de detalle paginada con badges de estado y exportación a Excel independiente por sección.

---

## Ruta

| Ruta                          | Guard      | Parámetros de URL |
| ------------------------------ | ----------- | ------------------ |
| `/logistica/guias-devoluciones` | `AuthGuard` | —                  |

---

## Propiedades clave

> Casi todas son getters que leen `DevolucionesStore` (signals) — el componente no mantiene su propio array de datos.

| Propiedad             | Tipo                | Descripción                                                              |
| ---------------------- | -------------------- | -------------------------------------------------------------------------- |
| `isLoading`            | `boolean`            | Activa skeleton del chart y deshabilita los botones de exportar            |
| `historicoListo`       | `boolean`            | `true` cuando los 2 workers históricos terminaron                          |
| `chartData`            | `DayPoint[]`         | Barras publicadas por `agregacion.worker`                                  |
| `tablaResumen`         | `TablaFila[]`        | Árbol jerárquico mes → fecha → tienda → guía                               |
| `tablaDetallePagina`   | `DevolucionRow[]`    | Página actual (50 filas) de la tabla de detalle                            |
| `filters`              | `FilterState[]`      | Estado de los 7 filtros, ahora en el Store                                 |

---

## Servicios y endpoints

| Servicio                | Método                       | Endpoint                          | Cuándo                              |
| ------------------------ | ----------------------------- | ---------------------------------- | ------------------------------------- |
| `DevolucionesRepository` | `getPrimeraPagina()`          | `GET metodoGenerico?coleccion=PedidosInter&...` | Fases 1 y 2 de carga    |
| `DevolucionesRepository` | `getTiendas()`                | `GET metodoGenerico?coleccion=Tienda&...`       | Al inicializar          |
| `DevolucionesStore`      | Todos los métodos             | —                                   | Al inicializar y en cada interacción |
| `agregacion.worker`      | `postMessage`                 | —                                   | Cada vez que cambian datos/filtros/página |

---

## Flujo principal

```
ngOnInit()
  -> store.inicializarFiltroMes()
  -> iniciarAggWorker()
  -> Promise.all([cargarDatos(), cargarTiendas()])

cargarDatos()
  -> Fase 1: repo.getPrimeraPagina() x7 estados -> store.appendLote() -> dispararAggregation()
  -> Fase 2: páginas restantes en segundo plano
  -> Fase 3: 2 workers de histórico (últimos 4 meses) -> store.setHistoricoListo(true)
```

---

## Historial de cambios

| Fecha      | Autor              | Cambio                                                                                                                    |
| ----------- | -------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| 2026-07-28 | Adalberto González | Migrado a Store + Repository + Rules + Workers (`helpers/`); histórico reducido de 8 a 4 meses; exportación a Excel dividida por panel; rediseño visual siguiendo el estándar de `dashboard-sin-despacho` |

---

## Observaciones

- El componente no contiene lógica de filtros ni de cómputo — todo se delega a `DevolucionesStore`, `devoluciones.rules.ts` y `agregacion.worker`.
- `estadoBadge()` y `tiendaCorta()` son wrappers delgados sobre funciones puras de `devoluciones.rules.ts`, expuestos como métodos para poder llamarlos desde el template.
