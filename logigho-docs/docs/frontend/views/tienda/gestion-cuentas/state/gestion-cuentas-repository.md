## Servicio: GestionCuentasRepository

**Ubicación:** `src/app/views/tienda/gestion-cuentas/state/gestion-cuentas.repository.ts`  
**Scope:** `providedIn: 'root'`

---

### ¿Qué hace?

Capa de acceso a datos del módulo. Es el único archivo que llama a `ConsumoGenericoService` y descomprime respuestas Zstd con `DecompressionService`. No toca el store ni contiene lógica de negocio.

---

### Métodos

| Método | Colección | Descripción |
|---|---|---|
| `getCuentas()` | `PancakeCuentasPrincipales` | Lista completa de cuentas |
| `crearCuenta(dto)` | `PancakeCuentasPrincipales` | Inserta documento nuevo |
| `editarCuenta(id, dto)` | `PancakeCuentasPrincipales` | Actualiza campos parciales por `_id` |
| `toggleEstado(id, estadoActual, fechaActualizacion)` | `PancakeCuentasPrincipales` | Invierte el estado (`Activo` ⇄ `Inactivo`) + actualiza `fechaActualizacion` |

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-15 | Adalberto González | Creación con las operaciones CRUD del módulo, todas sobre `PancakeCuentasPrincipales` |

---

### Observaciones

- Todas las lecturas pasan por `DecompressionService.decompressZstd()` (`mcomp=2` en la URL).
