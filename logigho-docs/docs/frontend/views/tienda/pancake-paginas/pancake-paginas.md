# Módulo: Pancake Páginas

---

## Autor: Adalberto González
Fecha creación: 2026-07-24  
Estado: desarrollo  
Tipo: módulo (1 vista + 1 componente hijo + 1 repositorio + 1 utilidad)

---

## Índice

1. [Vista: PancakePaginasComponent](#1-vista-pancakepaginascomponent)
2. [Servicio: PaginasRepository](repository/paginas-repository.md)
3. [Componente: PaginaDetalleDrawerComponent](components/pagina-detalle-drawer.md)
4. [Utilidad: resolver-corte](utils/resolver-corte.md)

---

## 1. Vista: PancakePaginasComponent

**Selector:** `app-pancake-paginas`  
**Ubicación:** `src/app/views/tienda/pancake-paginas/pancake-paginas.component.ts`  
**Acceso:** Autenticado

---

### ¿Qué hace?

Es el dashboard de desempeño de las páginas de Pancake: gasto, alcance y resultados de las campañas de WhatsApp Business y Facebook. Al abrirla, carga en paralelo las páginas y las estadísticas de campañas, y muestra:

- **5 tarjetas KPI**: total de páginas, activas, con error, gasto de hoy y páginas sin campañas activas.
- **Una tabla** con cada página: nombre, teléfono, usuarios activos, conexión, cantidad de campañas activas, gasto y estado de corte.
- **Filtros**: búsqueda por nombre/teléfono, y dropdowns de selección múltiple por página, plataforma, fecha, corte del día y conexión.
- Al hacer clic en una fila se "enfoca" esa página: la tabla, los KPIs, el gráfico y la tabla de comparación se recalculan solo con sus datos.
- Al hacer clic en el botón de detalle de una fila se abre un cajón lateral (`PaginaDetalleDrawerComponent`) con sus campañas, usuarios y productos asociados.
- Un gráfico de barras compara gasto, alcance o costo por resultado entre los distintos cortes del día.
- Una tabla de comparación detallada, anuncio por anuncio y por corte, exportable a Excel.

---

### Cortes del día (slots)

Los datos de gasto y resultados de un anuncio llegan varias veces al día, identificados por un `slotId`:

| Slot | Significado |
|---|---|
| `2` | Corte de las 9:00 AM (parcial) |
| `3` | Corte de las 2:00 PM (parcial) |
| `4` | Corte de las 5:00 PM (parcial) |
| `1` | Cierre oficial del día anterior |
| `99` / `5` | Verificación manual |

Para un mismo anuncio puede haber varias filas de un mismo día en distintos cortes. La pantalla siempre resuelve cuál es el dato vigente con esta prioridad: cierre oficial (`1`) > verificación manual (`99`/`5`) > corte parcial más reciente (`2`/`3`/`4`). Esa lógica está en [resolver-corte.ts](utils/resolver-corte.md), no en el componente.

---

### Ruta

```
tienda/pancake-paginas
```

---

### Decoradores y configuración técnica

```typescript
@Component({
  selector: 'app-pancake-paginas',
  standalone: true,
  imports: [CommonModule, FormsModule, PaginaDetalleDrawerComponent, TourGuiadoComponent],
  templateUrl: './pancake-paginas.component.html',
  styleUrl: './pancake-paginas.component.scss',
})
export class PancakePaginasComponent implements OnInit
```

> Igual que `gestion-paginas`, este módulo no tiene Store aparte: filtros, KPIs y resolución de cortes viven directamente en el componente usando `signal()` y `computed()`.

---

### Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `loading` / `error` | `Signal<boolean>` / `Signal<string \| null>` | Estado de carga y último error |
| `filtros` | `Signal<PaginasFiltros>` | Búsqueda, plataformas, páginas, conexiones, fechas y slots seleccionados |
| `paginaEnfocada` | `Signal<string \| null>` | `paginaId` de la página "enfocada" al hacer clic en su fila (cross-filtro de todo el dashboard) |
| `paginasFiltradas` / `paginasPagina` / `totalPaginas` | `computed` | Resultado final de la tabla principal, con paginación |
| `kpis` | `computed` | `{ totalPaginas, paginasActivas, paginasConError, gastoTotalHoy, alcanceTotalHoy, paginasSinCampanas }` |
| `fechasDisponibles` / `fechasEnUso` | `computed` | Fechas que realmente tienen datos, y las que están efectivamente en uso (elegidas o la más reciente por defecto) |
| `slotsConDatos` | `computed` | Filtra los slots `99`/`5` cuando nunca han tenido gasto o resultados reales, para no ensuciar el filtro con opciones vacías |
| `graficoAgrupado` / `graficoValorMax` | `computed` | Datos ya armados para pintar el gráfico de barras agrupado por corte |
| `filasComparacion` / `filasComparacionPagina` / `totalPaginasComparacion` | `computed` | Tabla de comparación detallada, anuncio por anuncio y por corte, con su propia paginación |
| `campanasDeSeleccionada` | `computed` | Campañas resueltas de la página actualmente abierta en el drawer |
| `drawerAbierto` | `boolean` | Controla la visibilidad del cajón lateral de detalle |
| `metricaGrafico` | `Signal<'gasto' \| 'alcance' \| 'costoPorResultado'>` | Métrica elegida para el gráfico único |

---

### Servicios y endpoints

| Servicio | Uso |
|---|---|
| `PaginasRepository` | Lee las colecciones `PancakePaginas` (páginas + productos asociados) y `PancakeEstadisticasPaginas` (filas de estadísticas por anuncio/corte/fecha) |

Ver detalle completo en [Repositorio](repository/paginas-repository.md).

---

### Flujo principal

```
ngOnInit()
  └─► cargar()
        ├─► repo.getPaginas()       → _paginasRaw.set(...)
        └─► repo.getEstadisticas()  → _estadisticasRaw.set(...)     (en paralelo)

Usuario escribe en el buscador
  └─► onBusqueda(valor) → setFiltros({ busqueda })

Usuario hace clic en una fila de la tabla
  └─► toggleEnfoque(paginaId)
        └─► Si ya estaba enfocada, la desenfoca; si no, la enfoca
              └─► Todo el dashboard (KPIs, gráficos, tabla de comparación) se recalcula solo para esa página

Usuario hace clic en el botón de "ver detalle" (lupa) de una fila
  └─► abrirDrawer(pagina)
        └─► drawerAbierto = true, _paginaSeleccionada.set(pagina)
              └─► <app-pagina-detalle-drawer> muestra campañas, usuarios y productos de esa página

Usuario cambia la métrica del gráfico (Gasto / Alcance / Costo por resultado)
  └─► setMetricaGrafico(metrica) → graficoAgrupado se recalcula

Usuario exporta la comparación de cortes
  └─► exportarComparacionExcel() → genera un .xlsx con la librería `xlsx`, respetando los filtros activos
```

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-24 | Adalberto González | Documentación y creación inicial del módulo, ya en su forma actual (con drawer de detalle, resolución de cortes y comparación exportable a Excel) |

---

### Observaciones

- Este módulo es puramente de **lectura y análisis** — a diferencia de `gestion-paginas`, aquí no se edita nada directamente sobre las páginas ni sus campañas. La única escritura relacionada (asociar productos) se hace desde `gestion-paginas`; esta pantalla solo la **muestra** en el drawer.
- El "enfoque" (`toggleEnfoque`) es distinto a la selección del drawer (`_paginaSeleccionada`): el enfoque filtra **toda la pantalla** (tabla, KPIs, gráficos), mientras que la selección del drawer solo controla qué se muestra en el cajón lateral. Se puede tener una página enfocada y abrir el drawer de otra distinta.
- Las filas con `status: 'error'` de `PancakeEstadisticasPaginas` (llegan con casi todos los campos en `null`, cuando falló la sincronización con Pancake) se excluyen de las sumas de gasto/alcance, pero sí se usan para mostrar el badge de "Error" en la tabla.
