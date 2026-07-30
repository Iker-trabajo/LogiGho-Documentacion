# Módulo: Pancake Páginas

---

## Autor: Adalberto González
Fecha creación: 2026-07-24
Última actualización: 2026-07-30
Estado: desarrollo
Tipo: módulo (1 vista + 2 componentes hijos + carpeta `helpers/` con repositorio, pipeline de estadísticas, export a Excel y resolución de cortes)

---

## Índice

1. [Vista: PancakePaginasComponent](#1-vista-pancakepaginascomponent)
2. [Clase: EstadisticasPipeline](helpers/estadisticas-pipeline.md)
3. [Servicio: PaginasRepository](helpers/paginas-repository.md)
4. [Servicio: ExportExcelService](helpers/export-excel-pipeline.md)
5. [Utilidad: resolver-corte](helpers/resolver-corte.md)
6. [Componente: PaginaDetalleDrawerComponent](components/pagina-detalle-drawer.md)
7. [Componente: PaginasErrorModalComponent](components/paginas-error-modal.md)

---

## 1. Vista: PancakePaginasComponent

**Selector:** `app-pancake-paginas`
**Ubicación:** `src/app/views/tienda/pancake-paginas/pancake-paginas.component.ts`
**Acceso:** Autenticado

---

### ¿Qué hace?

Es el dashboard de desempeño de las páginas de Pancake: gasto, alcance y resultados de las campañas de WhatsApp Business y Facebook. Al abrirla, carga en paralelo las páginas, las estadísticas de campañas y las cuentas principales, y muestra:

- **5 tarjetas KPI**: total de páginas, activas, con error, gasto de hoy y páginas sin campañas activas.
- **Una tabla** con cada página: nombre, teléfono, usuarios activos, conexión, cantidad de campañas activas, gasto y estado de corte.
- **Filtros**: búsqueda por nombre/teléfono, y dropdowns de selección múltiple por página, plataforma, fecha, corte del día, conexión y red de anuncios (Meta/TikTok).
- Al hacer clic en una fila se "enfoca" esa página: la tabla, los KPIs, el gráfico y la tabla de comparación se recalculan solo con sus datos.
- Al hacer clic en el botón de detalle de una fila se abre un cajón lateral (`PaginaDetalleDrawerComponent`) con sus campañas, usuarios y productos asociados.
- Al hacer clic en la tarjeta KPI **"Páginas con error"** se abre un modal (`PaginasErrorModalComponent`) que lista todas las páginas con error de conexión o de sincronización.
- Un gráfico de barras compara gasto, alcance o costo por resultado entre los distintos cortes del día.
- Una tabla de comparación detallada, anuncio por anuncio y por corte, exportable a Excel (resumen por página o detalle fila por fila).

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

Para un mismo anuncio puede haber varias filas de un mismo día en distintos cortes. La pantalla siempre resuelve cuál es el dato vigente con esta prioridad: cierre oficial (`1`) > verificación manual (`99`/`5`) > corte parcial más reciente (`2`/`3`/`4`). Esa lógica está en [resolver-corte.ts](helpers/resolver-corte.md), consumida por [EstadisticasPipeline](helpers/estadisticas-pipeline.md) — no directamente en el componente.

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
  imports: [CommonModule, FormsModule, PaginaDetalleDrawerComponent, PaginasErrorModalComponent, TourGuiadoComponent],
  templateUrl: './pancake-paginas.component.html',
  styleUrl: './pancake-paginas.component.scss',
})
export class PancakePaginasComponent implements OnInit
```

> Igual que `gestion-paginas`, este módulo no tiene Store aparte: filtros, KPIs y paginación viven directamente en el componente usando `signal()` y `computed()`. Lo que sí cambió respecto a la versión inicial: el **cálculo** de estadísticas (raw → resumen por página/campaña → agregados) ya no vive en el componente, sino en una clase aparte, [`EstadisticasPipeline`](helpers/estadisticas-pipeline.md), instanciada una vez en el constructor de la propiedad `stats`. El componente consume sus `computed` (`this.stats.paginas()`, `this.stats.slotVigenteGlobal()`, etc.) igual que si fueran propios.

---

### Estructura de archivos del módulo

```
pancake-paginas/
├── components/
│   ├── pagina-detalle-drawer/       → drawer de detalle (ver doc)
│   └── paginas-error-modal/         → modal del KPI "Páginas con error" (ver doc)
├── helpers/
│   ├── estadisticas-pipeline.ts     → cálculo: raw → resumen → agregados
│   ├── export-excel.pipeline.ts     → generación de los .xlsx
│   ├── paginas.repository.ts        → acceso a datos (HTTP)
│   ├── resolver-corte.ts            → funciones puras de resolución de cortes
│   └── resolver-corte.spec.ts
├── models/
│   └── paginas.models.ts            → interfaces, tipos y constantes del módulo
├── pancake-paginas.component.ts
├── pancake-paginas.component.html
├── pancake-paginas.component.scss
└── pancake-paginas.component.spec.ts
```

Todo lo que no es la vista misma (acceso a datos, cálculo, exportación, funciones puras) vive centralizado en `helpers/`, sin subcarpetas — es deliberado: son pocos archivos y no ameritan más anidamiento.

---

### Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `loading` / `error` | `Signal<boolean>` / `Signal<string \| null>` | Estado de carga y último error |
| `filtros` | `Signal<PaginasFiltros>` | Búsqueda, plataformas, páginas, conexiones, fechas, slots y redes de anuncios seleccionados |
| `paginaEnfocada` | `Signal<string \| null>` | `paginaId` de la página "enfocada" al hacer clic en su fila (cross-filtro de todo el dashboard) |
| `paginasFiltradas` / `paginasPagina` / `totalPaginas` | `computed` | Resultado final de la tabla principal, con paginación |
| `kpis` | `computed` | `{ totalPaginas, paginasActivas, paginasConError, gastoTotalHoy, alcanceTotalHoy, paginasSinCampanas }` |
| `paginasConError` | `computed` | Páginas con `codigoError` o `errorSincronizacion`, dentro del alcance de filtros/enfoque activo — fuente del modal del KPI "Páginas con error" |
| `modalErrorAbierto` | `boolean` | Controla la visibilidad del modal de páginas con error |
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
| `PaginasRepository` | Lee las colecciones `PancakePaginas` (páginas + productos asociados), `PancakeEstadisticasPaginas` (filas de estadísticas por anuncio/corte/fecha) y `PancakeCuentasPrincipales` (cuentas madre, para el "Perfil Admin") |
| `ExportExcelService` | Genera los `.xlsx` de resumen y detalle a partir de datos ya calculados por el componente |

Ver detalle completo en [PaginasRepository](helpers/paginas-repository.md) y [ExportExcelService](helpers/export-excel-pipeline.md).

---

### Flujo principal

```
ngOnInit()
  └─► cargar()
        ├─► repo.getPaginas()             → _paginasRaw.set(...)
        ├─► repo.getEstadisticas()        → _estadisticasRaw.set(...)         (en paralelo)
        └─► repo.getCuentasPrincipales()  → _cuentasPrincipalesRaw.set(...)   (en paralelo)

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

Usuario hace clic en la tarjeta KPI "Páginas con error"
  └─► abrirModalError() → modalErrorAbierto = true
        └─► <app-paginas-error-modal> lista paginasConError()
              └─► Al elegir una página → verPaginaConError(pagina) → solo abrirDrawer(pagina)
                    (NO se enfoca la página ni se filtra el resto del dashboard)

Usuario cambia la métrica del gráfico (Gasto / Alcance / Costo por resultado)
  └─► setMetricaGrafico(metrica) → graficoAgrupado se recalcula

Usuario exporta la comparación de cortes
  └─► exportarComparacionExcel() / exportarDetalleExcel()
        └─► arma los parámetros desde sus `computed` y llama a ExportExcelService, respetando los filtros activos
```

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-30 | Adalberto González | **Refactor de estructura**: se extrajo todo el pipeline de cálculo de estadísticas del componente a la nueva clase [`EstadisticasPipeline`](helpers/estadisticas-pipeline.md) y la exportación a Excel al servicio [`ExportExcelService`](helpers/export-excel-pipeline.md), reduciendo el componente de ~940 a ~550 líneas sin cambiar su comportamiento. Se reubicaron `repository/` y `utils/` dentro de una única carpeta `helpers/`, sin subcarpetas. **Nueva funcionalidad**: se agregó [`PaginasErrorModalComponent`](components/paginas-error-modal.md), un modal que lista las páginas con error al hacer clic en la tarjeta KPI "Páginas con error". De paso se corrigió ese mismo KPI: antes solo contaba `codigoError` (problemas de conexión/token), ahora también cuenta `errorSincronizacion` (fallos al sincronizar estadísticas de Pancake), que antes se veían en el badge de la tabla pero no se sumaban al KPI |
| 2026-07-24 | Adalberto González | Documentación y creación inicial del módulo, ya en su forma actual (con drawer de detalle, resolución de cortes y comparación exportable a Excel) |

---

### Observaciones

- Este módulo es puramente de **lectura y análisis** — a diferencia de `gestion-paginas`, aquí no se edita nada directamente sobre las páginas ni sus campañas. La única escritura relacionada (asociar productos) se hace desde `gestion-paginas`; esta pantalla solo la **muestra** en el drawer.
- El "enfoque" (`toggleEnfoque`) es distinto a la selección del drawer (`_paginaSeleccionada`): el enfoque filtra **toda la pantalla** (tabla, KPIs, gráficos), mientras que la selección del drawer solo controla qué se muestra en el cajón lateral. Se puede tener una página enfocada y abrir el drawer de otra distinta. El modal de páginas con error sigue la misma regla que el drawer: elegir una página ahí **no** la enfoca.
- Las filas con `status: 'error'` de `PancakeEstadisticasPaginas` (llegan con casi todos los campos en `null`, cuando falló la sincronización con Pancake) se excluyen de las sumas de gasto/alcance, pero sí se usan para mostrar el badge de "Error" en la tabla y para contar en el KPI "Páginas con error" y en el modal correspondiente.
- Una página puede tener error de conexión (`codigoError`, p. ej. token de acceso vencido) y/o error de sincronización (`errorSincronizacion`) al mismo tiempo; el modal y el KPI cuentan la página una sola vez si tiene cualquiera de los dos, y el modal prioriza mostrar `codigoError` cuando ambos están presentes.
