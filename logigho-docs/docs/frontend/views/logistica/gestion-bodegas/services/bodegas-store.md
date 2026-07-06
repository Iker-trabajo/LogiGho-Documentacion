## Store: BodegasStore

**Ubicación:** `src/app/views/logistica/gestion-bodegas/store/bodegas.store.ts`
**Scope:** `providedIn: 'root'`

---

### ¿Qué hace?

Gestiona la lista de bodegas, los filtros activos, la paginación y los KPIs derivados.

---

### Signals y computed

| Signal / Computed | Tipo | Descripción |
|---|---|---|
| `bodegas` | `Signal<Bodega[]>` | Lista completa, ordenada desc por `fechaCreacion` |
| `loading` | `Signal<boolean>` | `true` mientras se espera respuesta del backend |
| `error` | `Signal<string \| null>` | Mensaje de error, o `null` si todo está bien |
| `filtrosBodega` | `Signal<BodegaFiltros>` | Búsqueda de texto y filtro de estado activos |
| `pagina` | `Signal<number>` | Página actual (base 1) |
| `bodegasFiltradas` | `computed` | Aplica `filtrosBodega` sobre `bodegas` |
| `bodegasPagina` | `computed` | Slice de la página actual sobre `bodegasFiltradas` |
| `totalPaginas` | `computed` | Páginas totales según el resultado filtrado |
| `kpis` | `computed` | `{ totalBodegas, bodegasActivas }` derivados de la lista completa |

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-06-24 | Adalberto González | Creación del store con signals, filtros, paginación y KPIs derivados |

---

### Observaciones

- `setBodegas()` ordena siempre por `fechaCreacion` descendente al recibir datos del backend.
- `setFiltrosBodega()` resetea la paginación a página 1 automáticamente al cambiar cualquier filtro.
- El tamaño de página es `PAGE_SIZE = 5`, constante privada del store.
