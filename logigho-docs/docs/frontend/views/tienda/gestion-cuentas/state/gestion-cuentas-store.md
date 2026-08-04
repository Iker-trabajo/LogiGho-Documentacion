## Store: GestionCuentasStore

**Ubicación:** `src/app/views/tienda/gestion-cuentas/state/gestion-cuentas.store.ts`  
**Scope:** `providedIn: 'root'`

---

### ¿Qué hace?

Gestiona la lista de cuentas, los filtros activos, la paginación y los KPIs derivados. Nunca hace HTTP ni conoce al `Repository`.

---

### Signals y computed

| Signal / Computed | Tipo | Descripción |
|---|---|---|
| `cuentas` | `Signal<Cuenta[]>` | Lista completa, ordenada desc por `cuentaId` numérico |
| `loading` | `Signal<boolean>` | `true` mientras se espera respuesta del backend |
| `error` | `Signal<string \| null>` | Mensaje de error, o `null` si todo está bien |
| `filtros` | `Signal<CuentasFiltros>` | Búsqueda de texto y filtro de estado activos |
| `pagina` | `Signal<number>` | Página actual (base 1) |
| `pageSize` | `number` | Tamaño de página fijo (`PAGE_SIZE = 5`) |
| `cuentasFiltradas` | `computed` | Aplica `filtros` sobre `cuentas` (nombre o `cuentaId`) |
| `cuentasPagina` | `computed` | Slice de la página actual sobre `cuentasFiltradas` |
| `totalPaginas` | `computed` | Páginas totales según el resultado filtrado |
| `kpis` | `computed` | `{ total, activas, inactivas }` derivados de la lista completa |

---

### Métodos

| Método | Descripción |
|---|---|
| `existeCuentaId(cuentaId)` | `true` si ya existe una cuenta con ese ID |
| `siguienteCuentaId()` | Calcula el máximo `cuentaId` numérico + 1 (arranca en `10001` si no hay cuentas) |
| `setCuentas(cuentas)` | Reemplaza el listado completo, ya ordenado |
| `upsertCuenta(cuenta)` | Inserta o reemplaza por `_id`, y reordena todo el listado |
| `removeCuenta(id)` | Elimina una cuenta por `_id` |
| `setFiltros(parcial)` | Actualiza filtros y resetea la página a 1 |
| `setPagina(n)` / `setLoading(v)` / `setError(msg)` | Setters directos de estado |
| `reset()` | Restablece el estado inicial (usado en `ngOnDestroy` del componente) |

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-15 | Adalberto González | Creación del store con signals, filtros, paginación y KPIs derivados |

---

### Observaciones

- `setFiltros()` resetea la paginación a página 1 automáticamente al cambiar cualquier filtro, igual que en `BodegasStore`.
