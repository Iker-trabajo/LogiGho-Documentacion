## Clase: EstadisticasPipeline

**Ubicación:** `src/app/views/tienda/pancake-paginas/helpers/estadisticas-pipeline.ts`
**Tipo:** clase plana con `computed()` de Angular, **no** `@Injectable`

---

### ¿Qué hace?

Concentra todo el pipeline de derivación de estadísticas del módulo: parte de las páginas y estadísticas crudas (tal como las entrega [PaginasRepository](paginas-repository.md)) y las va transformando, paso a paso, hasta llegar a lo que la vista necesita pintar — el resumen por página, las campañas agregadas, los datos del gráfico y las filas de la tabla de comparación.

Nace de extraer del componente `PancakePaginasComponent` los ~15 `computed` que antes vivían ahí mezclados con el estado de filtros y dropdowns. No es un servicio de Angular (`@Injectable`): se instancia **una sola vez**, en el constructor de la propiedad `stats` del componente, pasándole los signals del propio componente por parámetro. Angular no necesita inyectarla porque no tiene dependencias externas — solo lee los signals que recibe.

---

### Instanciación

```typescript
private readonly stats: EstadisticasPipeline = new EstadisticasPipeline(
  this._paginasRaw,
  this._estadisticasRaw,
  this._filtros,
  this.pageIdsActivos,
  this.fechasEnUso,
  this.coloresFecha,
);
```

El componente le pasa sus propios signals (páginas y estadísticas crudas, filtros, IDs de página activos según los filtros base, fechas en uso y la paleta de colores del gráfico) y luego consume los `computed` de la instancia (`this.stats.paginas()`, `this.stats.kpis`, etc.) exactamente igual que si fueran propios.

---

### Propiedades y métodos clave

| Propiedad/método | Tipo | Descripción |
|---|---|---|
| `fechasDisponibles` | `computed` | Fechas con estadísticas registradas, más reciente primero |
| `estadisticasEnAlcance` | `computed` | Estadísticas de páginas activas, sin filas `status: 'error'`, filtradas por red de anuncios — base compartida por todo el resto del pipeline |
| `erroresSincronizacion` | `computed` | Último `errorDetail` de sincronización por página, indexado por `pageId` |
| `estadisticasPorFecha` | `computed` | Estadísticas acotadas a las fechas seleccionadas, o a la más reciente si no hay ninguna |
| `filasComparacion` | `computed` | Una fila por cada combinación anuncio+fecha+slot seleccionada, **sin** resolver a una sola vigente (a diferencia de `anunciosResueltos`). Fuente de la tabla de comparación de cortes |
| `anunciosResueltos` | `computed` | Una fila vigente por anuncio, resuelta con [`resolverFilaVigente`](resolver-corte.md) |
| `slotVigenteGlobal` | `computed` | Slot vigente único para todo el conjunto (no por anuncio). Prioridad: cierre (`1`) > verificación manual (`99`/`5`) > mayor intradía disponible (`2`/`3`/`4`) |
| `gastoRealPorPagina` | `computed` | Gasto y resultados de "hoy" por página, del `slotVigenteGlobal` únicamente |
| `campanasPorPagina` | `computed` | Campañas agregadas por página, indexadas por `pageId` → `campanaId` |
| `paginas` | `computed` | Páginas enriquecidas con usuarios, campañas, gasto/resultados de hoy, estado de corte y error de sincronización — el `PaginaResumen[]` que consume la tabla principal |
| `slotsConDatos` | `computed` | Slots disponibles, excluyendo `99`/`5` (verificación manual) cuando no tienen datos |
| `graficoAgrupado(metrica)` | método | Grupos por slot con una barra por fecha, para la métrica elegida (gasto, alcance o costo por resultado) |
| `metricasPorPaginaYRed` | `computed` | Gasto y mensajes del slot vigente, por página y desglosados por red (Meta/TikTok) — fuente del export de resumen |

También exporta la función auxiliar `aNumero(valor)`, que convierte los strings numéricos crudos del backend a `number` (o `0` si no es un número válido).

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-30 | Adalberto González | Extracción de todo el pipeline de estadísticas desde `PancakePaginasComponent` (~15 `computed`) a esta clase, para reducir el componente de ~940 a ~550 líneas sin cambiar su comportamiento. El componente pasó de calcular todo directamente a instanciar `EstadisticasPipeline` una vez y delegarle el cálculo |

---

### Observaciones

- El único acoplamiento con Angular es el uso de `computed()` — internamente sigue exactamente el mismo patrón reactivo que tenía el componente antes de la extracción, así que el recálculo perezoso y la memoización se comportan igual.
- `graficoAgrupado` es un **método**, no un `computed`, porque necesita recibir la métrica elegida (`'gasto' | 'alcance' | 'costoPorResultado'`) como parámetro — el componente lo envuelve en su propio `computed(() => this.stats.graficoAgrupado(this._metricaGrafico()))` para que sí reaccione a cambios de métrica.
- La corrección del cross-filtro por `pageIdsActivos` sigue viviendo en el componente (`cumpleFiltrosBase`), no aquí: `EstadisticasPipeline` recibe ese `Set<string>` ya resuelto y no conoce la lógica de filtrado de búsqueda/plataforma/conexión.
