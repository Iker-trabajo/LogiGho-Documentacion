## Servicio: BodegasRepository

**Ubicación:** `src/app/views/logistica/gestion-bodegas/repository/bodegas.repository.ts`
**Scope:** `providedIn: 'root'`

---

### ¿Qué hace?

Capa de acceso a datos del módulo. Es el único archivo que llama a `ConsumoGenericoService` y descomprime respuestas GZIP con `DecompressionService`. No toca el store ni contiene lógica de negocio.

---

### Métodos

| Método | Colección | Descripción |
|---|---|---|
| `getBodegas()` | `Bodegas` | Lista completa de bodegas |
| `crearBodega(dto)` | `Bodegas` | Inserta documento nuevo |
| `editarBodega(id, dto)` | `Bodegas` | Actualiza campos parciales por `_id` |
| `getCiudades()` | `Ciudades` | Lista ordenada con formato `"Departamento - Ciudad"` |
| `getProductos(tienda)` | `Productos` | Catálogo filtrado por tienda, ordenado alfabéticamente |
| `getTodasLasAsignaciones()` | `AsignacionBodegas` | Todas las asignaciones sin filtro |
| `getAsignacionesPorBodega(idBodega)` | `AsignacionBodegas` | Asignaciones de una bodega específica |
| `crearAsignacion(dto)` | `AsignacionBodegas` | Inserta nueva asignación |
| `actualizarAsignacion(id, estado)` | `AsignacionBodegas` | Cambia estado + `fechaActualizacion` |
| `eliminarAsignacion(id)` | `AsignacionBodegas` | Elimina por `_id` |
| `getStockPorProducto()` | `ResumenInventario` | Devuelve `Map<idProducto, Actual>` sin filtro de tienda |

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-06-24 | Adalberto González | Creación con todas las operaciones CRUD del módulo |
| 2026-06-25 | Adalberto González | Migradas todas las consultas de GZIP a Zstd (`mcomp=2`) |

---

### Observaciones

- Todas las respuestas del backend pasan por `DecompressionService.decompressGzip()` antes de ser procesadas.
- `getStockPorProducto()` agrega el campo `Actual` por `IdProducto` en caso de que un mismo producto aparezca en múltiples registros de `ResumenInventario`.
