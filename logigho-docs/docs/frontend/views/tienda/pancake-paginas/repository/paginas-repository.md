## Servicio: PaginasRepository

**Ubicación:** `src/app/views/tienda/pancake-paginas/repository/paginas.repository.ts`  
**Scope:** `providedIn: 'root'`

---

### ¿Qué hace?

Capa de acceso a datos del módulo. Trae la lista de páginas (con usuarios y productos asociados) y las filas crudas de estadísticas de campañas por anuncio/corte/fecha. No decide cuál dato es el vigente ni hace ninguna cuenta — eso vive en el componente y en [resolver-corte](../utils/resolver-corte.md).

---

### Métodos

| Método | Colección | Descripción |
|---|---|---|
| `getPaginas()` | `PancakePaginas` | Lista completa de páginas, pidiendo al backend solo los campos declarados en `CAMPOS_PAGINA` (incluye `productosAsociados`) |
| `getEstadisticas()` | `PancakeEstadisticasPaginas` | Filas crudas de estadísticas de campañas, una por anuncio/corte/fecha, pidiendo solo los campos de `CAMPOS_ESTADISTICA` |

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-24 | Adalberto González | Documentación inicial. Se agregó `productosAsociados` a `CAMPOS_PAGINA` para que el drawer de detalle pueda mostrar qué productos despacha cada página (dato que ahora se gestiona desde el módulo `gestion-paginas`) |

---

### Observaciones

- Las lecturas pasan por `DecompressionService.decompressZstd()` (`mcomp=2` en la URL), igual que en el resto de módulos que usan `ConsumoGenericoService`.
- Este repositorio **no tiene métodos de escritura** — es solo lectura. Los productos asociados a una página se editan desde `GestionPaginasRepository` (módulo `gestion-paginas`); aquí solo se leen para mostrarlos.
- Las filas de `PancakeEstadisticasPaginas` llegan "crudas": todos los valores numéricos (gasto, alcance, clics, etc.) vienen como `string`, no como `number` — la conversión a número y la resolución de cuál fila es la vigente ocurren en el componente y en `resolver-corte.ts`, nunca en este repositorio.
