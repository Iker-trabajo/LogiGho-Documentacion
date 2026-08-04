## Servicio: ExportExcelService

**Ubicación:** `src/app/views/tienda/pancake-paginas/helpers/export-excel.pipeline.ts`
**Scope:** `providedIn: 'root'`

---

### ¿Qué hace?

Genera los dos archivos Excel del módulo, usando la librería `xlsx`. Ambos métodos son **puros**: no leen signals ni el estado del componente, reciben todo lo que necesitan como parámetros ya calculados. Se extrajeron de `PancakePaginasComponent` (donde antes eran `exportarComparacionExcel()` y `exportarDetalleExcel()`) para que el componente no cargue con la construcción de hojas y anchos de columna.

A pesar del nombre del archivo (`export-excel.pipeline.ts`, para agrupar visualmente junto a `estadisticas-pipeline.ts` en el listado de `helpers/`), esta sí es una clase `@Injectable` normal — a diferencia de `EstadisticasPipeline`, que se instancia manualmente.

---

### Métodos

| Método | Genera | Descripción |
|---|---|---|
| `exportarComparacion(paginasFiltradas, metricasPorPaginaId, nombresCuentasPorId)` | `resumen-paginas_YYYY-MM-DD.xlsx` | Resumen agregado por página/tienda (no por anuncio): gasto y mensajes de Meta/TikTok, producto asociado y "Perfil Admin" (cuenta madre) |
| `exportarDetalle(filasComparacion, nombrePorPageId, redesAnunciosLabels)` | `detalle-anuncios_YYYY-MM-DD.xlsx` | Detalle fila por fila (un anuncio/campaña por corte y fecha), con todos los campos que entrega el backend, incluidos los exclusivos de Meta (`ads`) o TikTok (`tt_ads`) |

El componente las invoca desde `exportarComparacionExcel()` y `exportarDetalleExcel()`, armando los parámetros a partir de sus propios `computed` (`paginasFiltradas()`, `this.stats.metricasPorPaginaYRed()`, etc.) antes de llamar al servicio.

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-30 | Adalberto González | Extracción de la lógica de exportación a Excel desde `PancakePaginasComponent` a este servicio, como parte de la reducción del componente principal. Documentación inicial |

---

### Observaciones

- Ninguno de los dos métodos toca el DOM ni dispara un HTTP: `XLSX.writeFile()` genera el archivo en memoria y dispara la descarga directamente desde el navegador.
- El nombre del archivo siempre incluye la fecha del día en que se exporta (`new Date().toISOString().slice(0, 10)`), no la fecha de los datos filtrados — si el usuario exporta hoy datos de una fecha pasada, el nombre del archivo igual dice la fecha de hoy.
- `exportarComparacion` resuelve el "Perfil Admin" con `nombresCuentasPorId`, un mapa que el componente arma a partir de `CuentaPrincipal[]` (ver [PaginasRepository](paginas-repository.md)); si la página no tiene `cuentaMadreId`, esa columna queda vacía.
