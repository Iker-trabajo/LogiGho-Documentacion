## Servicio: GestionPaginasRepository

**Ubicación:** `src/app/views/tienda/gestion-paginas/state/gestion-paginas.repository.ts`  
**Scope:** `providedIn: 'root'`

---

### ¿Qué hace?

Capa de acceso a datos del módulo. Es el único archivo que llama a `ConsumoGenericoService` y descomprime respuestas Zstd con `DecompressionService`. No toca el store ni contiene lógica de negocio.

---

### Métodos

| Método | Colección | Descripción |
|---|---|---|
| `getPaginas()` | `PancakePaginas` | Lista completa de páginas, pidiendo al backend solo los campos declarados en `CAMPOS_PAGINA` |

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-16 | Adalberto González | Creación con la única operación de lectura del módulo, sobre `PancakePaginas` |

---

### Observaciones

- La lectura pasa por `DecompressionService.decompressZstd()` (`mcomp=2` en la URL).
- El parámetro `campos=` (plural) le indica al backend exactamente qué campos devolver — se usa para excluir `tokenAccesoPagina` y `_id` desde el propio backend, por seguridad: ningún componente de presentación de este módulo llega a recibir esos campos.
