## Store: GestionPaginasStore

**Ubicación:** `src/app/views/tienda/gestion-paginas/state/gestion-paginas.store.ts`  
**Scope:** `providedIn: 'root'`

---

### ¿Qué hace?

Gestiona la lista de páginas, los filtros activos (búsqueda, multiselección de plataforma/conexión/suscripción/páginas y activación), el criterio de orden, la paginación y los KPIs derivados. Nunca hace HTTP ni conoce al `Repository`.

---

### Signals y computed

| Signal / Computed | Tipo | Descripción |
|---|---|---|
| `paginas` | `Signal<Pagina[]>` | Lista completa tal como llegó del backend |
| `loading` | `Signal<boolean>` | `true` mientras se espera respuesta del backend |
| `error` | `Signal<string \| null>` | Mensaje de error, o `null` si todo está bien |
| `filtros` | `Signal<PaginasFiltros>` | Búsqueda, arrays de multiselección y filtro de activación |
| `orden` | `Signal<PaginasOrden>` | Criterio de orden activo (`fechaActualizacion` por defecto; no editable desde la UI actual) |
| `pagina` | `Signal<number>` | Página actual de la tabla (base 1) |
| `pageSize` | `number` | Tamaño de página fijo (`PAGE_SIZE = 10`) |
| `plataformasDisponibles` | `computed` | Plataformas únicas presentes en `paginas`, ordenadas alfabéticamente — pobla el dropdown de Plataforma |
| `nombresDisponibles` | `computed` | Nombres de página únicos presentes en `paginas`, ordenados alfabéticamente — pobla el dropdown de Páginas |
| `paginasFiltradas` | `computed` | Aplica búsqueda + todos los filtros multiselección (AND entre campos, OR dentro de cada uno) + filtro de activación, y ordena según `orden` |
| `paginasPagina` | `computed` | Slice de la página actual sobre `paginasFiltradas` |
| `totalPaginas` | `computed` | Páginas totales según el resultado filtrado |
| `kpis` | `computed` | `{ total, activadas, conectadas, usuariosActivos }` derivados de la lista completa |
| `paginaSeleccionada` | `computed` | Página del maestro-detalle seleccionada por `paginaId` (mecanismo alterno al modal, ver Observaciones) |

---

### Métodos

| Método | Descripción |
|---|---|
| `setPaginas(paginas)` | Reemplaza el listado completo |
| `setFiltros(parcial)` | Actualiza cualquier subconjunto de `PaginasFiltros` y resetea la página a 1 |
| `setOrden(orden)` | Cambia el criterio de ordenamiento |
| `seleccionar(paginaId)` | Marca una página como seleccionada por id, o `null` para deseleccionar |
| `setPagina(n)` / `setLoading(v)` / `setError(msg)` | Setters directos de estado |
| `reset()` | Restablece el estado inicial, incluyendo `FILTROS_INICIALES` (usado en `ngOnDestroy` del componente) |

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-16 | Adalberto González | Creación del store con signals, filtros, paginación y KPIs derivados |

---

### Observaciones

- `setFiltros()` resetea la paginación a página 1 automáticamente al cambiar cualquier filtro.
