## Servicio: GestionPaginasRepository

**Ubicación:** `src/app/views/tienda/gestion-paginas/repository/gestion-paginas.repository.ts`  
**Scope:** `providedIn: 'root'`

---

### ¿Qué hace?

Capa de acceso a datos del módulo. Es el único archivo que llama a `ConsumoGenericoService` y descomprime respuestas Zstd con `DecompressionService`. No contiene lógica de filtros, paginación ni KPIs — eso vive en el componente.

---

### Métodos

| Método | Colección | Descripción |
|---|---|---|
| `getPaginas()` | `PancakePaginas` | Lista completa de páginas, pidiendo al backend solo los campos declarados en `CAMPOS_PAGINA` (incluye `_id`, necesario para poder actualizar después) |
| `getProductos()` | `Productos` | Catálogo completo de productos de la tienda, proyectado solo a `idproducto` y `nombre` (lo mínimo que necesita el selector) |
| `actualizarProductosAsociados(id, productos)` | `PancakePaginas` | Reemplaza por completo el campo `productosAsociados` del documento identificado por `_id` |

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-16 | Adalberto González | Creación con la única operación de lectura del módulo, sobre `PancakePaginas` |
| 2026-07-24 | Adalberto González | Se agrega `_id` a los campos consultados de `PancakePaginas` (antes se excluía por seguridad); se agregan `getProductos()` y `actualizarProductosAsociados()` para soportar la asociación de productos a una página |

---

### Observaciones

- Las lecturas pasan por `DecompressionService.decompressZstd()` (`mcomp=2` en la URL).
- El parámetro `campos=` (plural) le indica al backend exactamente qué campos devolver — se usa para excluir `tokenAccesoPagina` desde el propio backend, por seguridad.
- `getProductos()` recorta y limpia (`.trim()`) el nombre del producto al mapearlo, porque los documentos de la colección `Productos` suelen traer espacios en blanco al inicio o al final (dato "sucio" que viene de otro sistema).
- `actualizarProductosAsociados()` usa `ConsumoGenericoService.actualizarGenerico()`, que siempre filtra la actualización por `_id` (nunca por `paginaId` u otro campo) — por eso `_id` tuvo que agregarse al modelo `Pagina` y a `CAMPOS_PAGINA`, algo que antes se evitaba a propósito por seguridad. No hay forma de cambiar ese comportamiento sin modificar `ConsumoGenericoService`, que es compartido por todo el proyecto.
- El array `productosAsociados` que se guarda **reemplaza** completamente al anterior — no es un "agregar", es un "esto es lo que hay ahora". Por eso el modal siempre envía la lista completa de productos seleccionados, no solo los nuevos.
