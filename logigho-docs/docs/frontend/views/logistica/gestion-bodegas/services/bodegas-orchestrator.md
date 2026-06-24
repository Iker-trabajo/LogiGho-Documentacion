## Servicio: BodegasOrchestratorService

**Ubicación:** `src/app/views/logistica/gestion-bodegas/services/bodegas-orchestrator.service.ts`
**Scope:** `providedIn: 'root'`

---

### ¿Qué hace?

Es el coordinador central del módulo. Ningún componente hace HTTP directamente: todo pasa por aquí. Coordina el repositorio (HTTP) y el store (estado), y mantiene estado mutable de sesión como el mapa de stock por bodega, las asignaciones del modal activo y la lista de productos en caché.

---

### Métodos clave

| Método | Descripción |
|---|---|
| `iniciar()` | Carga paralela de bodegas, ciudades, asignaciones y stock. Calcula `stockPorBodega` |
| `crearBodega(dto)` | Inserta bodega nueva y refresca la lista desde el backend |
| `editarBodega(id, dto)` | Actualiza campos parciales y refresca la lista |
| `toggleEstado(bodega)` | Alterna `Activa` ↔ `Inactiva` llamando internamente a `editarBodega` |
| `cargarProductos()` | Carga catálogo de productos filtrado por tiendas del usuario. Con caché en memoria |
| `crearAsignacion(idBodega, idProducto, estado)` | Crea documento en `AsignacionBodegas` con fecha actual |
| `cargarAsignacionesBodega(idBodega)` | Carga asignaciones de una bodega específica en `asignacionesDrawer` |
| `toggleAsignacion(asignacion)` | Alterna estado de una asignación y actualiza la lista local sin recargar backend |
| `eliminarAsignacion(asignacion)` | Elimina asignación por `_id` y la quita de la lista local |
| `bodegasBajoStockCount(bodegas)` | Cuenta bodegas con stock < 5000, incluyendo las sin asignaciones (stock = 0) |
| `getBodegasBajoStockLista(bodegas)` | Devuelve el array de bodegas bajo el umbral para alimentar el modal KPI |

---

### Getters

| Getter | Descripción |
|---|---|
| `stockTotalBodegas` | Suma todos los valores del mapa `stockPorBodega` |

---

### Propiedades de estado

| Propiedad | Tipo | Descripción |
|---|---|---|
| `stockPorBodega` | `Map<string, number>` | `idBodega → stockTotal`, calculado al iniciar |
| `asignacionesDrawer` | `AsignacionBodega[]` | Asignaciones de la bodega abierta en el modal de gestión |
| `cargandoDrawer` | `boolean` | `true` mientras se cargan las asignaciones del modal |
| `productos` | `ProductoOpcion[]` | Catálogo de productos en caché (se carga una sola vez) |
| `ciudades` | `Ciudad[]` | Lista de ciudades para el formulario (se carga una sola vez) |

---

### Endpoints que consume

| Colección | Operación | Cuándo |
|---|---|---|
| `Bodegas` | GET / POST / PUT | Listar, crear, editar, cambiar estado |
| `Ciudades` | GET | Al iniciar, para el selector del formulario |
| `Productos` | GET filtrado por tienda | Al abrir modal de asignación o gestión (con caché) |
| `AsignacionBodegas` | GET / POST / PUT / DELETE | Todas las operaciones de asignaciones |
| `ResumenInventario` | GET sin filtro de tienda | Al iniciar, para calcular stock por bodega |

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-06-24 | Adalberto González | Creación con carga paralela, cálculo de stock por bodega y gestión de asignaciones |

---

### Observaciones

- `calcularStockPorBodega()` solo suma asignaciones con `estado === 'Activa'`. Las inactivas no contribuyen al stock.
- El umbral `UMBRAL_BAJO_STOCK_BODEGA = 5000` es una constante privada. Si se necesita configurable, moverla a un archivo de configuración del módulo.
- `getTiendaParam()` sigue usándose para `cargarProductos()`. Solo el cálculo de stock de bodegas es global (sin filtro de tienda).
