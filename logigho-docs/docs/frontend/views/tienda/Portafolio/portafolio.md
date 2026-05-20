---

## Autor: Adalberto González

Fecha creacion: 2026-04-20  
Estado: desarrollo  
Tipo: vista

# Vista: Portafolio

**Selector:** `app-portafolio`  
**Ubicación:** `src/app/views/tienda/portafolio`  
**Acceso:** Autenticado | Rol: Todos (eliminación restringida a `CEO` y `Desarrollador`)

---

## ¿Qué hace?

Catálogo de gestión de productos de la tienda. Permite al usuario listar, buscar, filtrar por texto, tienda, categoría, proveedor, tipo y estado; crear productos individuales o en lote, editar y eliminar. Muestra estadísticas de inventario en tiempo real.

---

## Ruta

| Ruta | Guard | Parámetros de URL |
| ---- | ----- | ----------------- |
| `/tienda/portafolio` | `AuthGuard` | — |

---

## Decoradores *(eliminar si no tiene)*

| Decorador | Nombre | Tipo | Descripción |
| --------- | ------ | ---- | ----------- |
| `@HostListener` | `document:click` | — | Cierra todos los menús desplegables de los filtros al hacer clic fuera de ellos. |

---

## Propiedades clave

| Propiedad | Tipo | Descripción |
| --------- | ---- | ----------- |
| `productos` | `Producto[]` | Lista completa cargada desde el backend. |
| `filteredProducts` | `Producto[]` | Lista resultado de aplicar todos los filtros activos. |
| `selectedEstadoFiltro` | `string` | Estado activo en los tabs (`Activo`, `Inactivo` o vacío para Todos). |
| `isCreProdModalOpen` | `boolean` | Visibilidad del modal de creación individual. |
| `isCreProdMasiModalOpen` | `boolean` | Visibilidad del modal de carga masiva. |
| `modalActualizarAbierto` | `boolean` | Visibilidad del modal de edición. |
| `productoParaEditar` | `Producto \| null` | Producto seleccionado para editar. |
| `rolesUsuario` | `string[]` | Roles del usuario leídos de `sessionStorage → roles_asignados`. |
| `buscarTienda / buscarCategoria / buscarProveedor` | `string` | Texto del buscador en cada dropdown de filtro. |
| `tiendasFiltradas / categoriasFiltradas / proveedoresFiltrados` | `any[]` | Opciones filtradas para cada dropdown searchable. |

---

## Servicios y endpoints

| Servicio | Método | Endpoint | Cuándo |
| -------- | ------ | -------- | ------ |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET metodoGenerico?coleccion=Productos&Tienda={tiendas}` | Al inicializar y tras crear/editar/eliminar |
| `ConsumoGenericoService` | `eliminarGenerico()` | `DELETE metodoGenerico?coleccion=Productos&_id={_id}` | Al eliminar un producto |
| `DecompressionService` | `decompressGzip()` | — | Descomprime la respuesta del backend |
| `GetObjectService` | `obtenerObjeto()` | S3 `imagenes-productos-logigho` | Al generar el catálogo PDF |

---

## Flujo principal

```
ngOnInit()
  → Lee roles_asignados de sessionStorage
  → fetchTableData()
      → Lee tiendas_asignadas de sessionStorage
      → GET metodoGenerico?coleccion=Productos&Tienda=...
      → decompressGzip(data.Resultado)
      → this.productos = resultado
      → this.filteredProducts = [...this.productos]
  → Construye dropdowns de tienda, categoría y proveedor (valores únicos de productos)

── Filtros ──
aplicarFiltros()
  → combina: searchTerm + tienda + categoría + proveedor + tipo + estado
  → filteredProducts = resultado
  → currentPage = 1

── Creación individual ──
openCreProdModal() → isCreProdModalOpen = true
onProductoCreado() → fetchTableData() + reconstruye dropdowns

── Carga masiva ──
openCreProdMasiModal() → isCreProdMasiModalOpen = true
onProductoCreadoMasivo() → fetchTableData() + reconstruye dropdowns

── Edición ──
abrirModalActualizar(producto) → productoParaEditar = producto, modalActualizarAbierto = true
despuesDeActualizar() → fetchTableData() + resetearFiltros()

── Eliminación ──
eliminarProducto(producto)
  → verifica rol CEO / Desarrollador
  → Swal confirmación
  → eliminarGenerico(_id)
  → fetchTableData() + reconstruye dropdowns + resetearFiltros()
```

---

## Historial de cambios

| Fecha | Autor | Cambio |
| ---------- | ------ | ------ |
| 2026-04-20 | Adalberto González | Corrección race condition en generación de IDs al crear productos en lote. |
| 2026-04-20 | Adalberto González | `fetchTableData()` refrescada al inicio de cada submit para reducir ventana de colisión. |
| 2026-04-20 | Adalberto González | Adición de `onProductoCreado()` para refrescar tabla y dropdowns tras creación exitosa. |
| 2026-04-20 | Adalberto González | `.trim()` en nombre de tienda al crear productos para evitar inconsistencias de filtro. |
| 2026-05-20 | Adalberto González | Integración del modal de carga masiva (`app-creaciones-masiva-productos`) con stepper de 3 pasos. |
| 2026-05-20 | Adalberto González | Patrón Template Method en `portafolio/importacion-template-method/`: `ImportacionMasivaBase` y `ProductoImportacionValidator`. |
| 2026-05-20 | Adalberto González | Validación de columnas del Excel antes de procesar filas; error descriptivo con columnas faltantes. |
| 2026-05-20 | Adalberto González | Columna `NombreCategoria` en Excel: el usuario elige por nombre, el sistema resuelve al ID internamente. |
| 2026-05-20 | Adalberto González | Regla 1 activa: `precioventa > precioproveedor`. Regla 2 activa: `IdTienda` en tiendas asignadas al usuario. |
| 2026-05-20 | Adalberto González | Race condition en IDs corregida: `id` e `idproducto` se recalculan con datos frescos de la BD antes de insertar cada producto. |
| 2026-05-20 | Adalberto González | Reemplazo de `ngx-select-dropdown` por searchable dropdowns propios en la barra de filtros. |
| 2026-05-20 | Adalberto González | Filtro por estado (`Activo` / `Inactivo`) con tabs visuales y contadores en la barra de filtros. |
| 2026-05-20 | Adalberto González | Reset visual automático de los tres searchable dropdowns al crear, actualizar o eliminar (`resetearFiltros()`). |
| 2026-05-20 | Adalberto González | Cierre de los tres modales (creación, edición, masivo) al hacer clic fuera del contenido. |
| 2026-05-20 | Adalberto González | Bloqueo del botón "Actualizar" si no hay cambios — snapshot `JSON.stringify` con getter `haycambios`. |
| 2026-05-20 | Adalberto González | Botón "Limpiar archivo" en modal masivo: permite reiniciar el paso 1 para corregir el Excel y volver a subir. |

---

## Observaciones

- **Inconsistencia de conteo:** Productos con `Tienda` con espacios extra (`"Nombre tienda "`) no son retornados por el filtro exacto del backend. El `.trim()` al crear previene nuevos casos, pero los registros existentes requieren migración en el backend.
- **Race condition de IDs (mitigada en frontend):** La reconsulta a la BD antes de cada inserción reduce la ventana de colisión, pero la solución definitiva es que el backend genere y garantice los IDs atómicamente.
- **Template Method en `portafolio/importacion-template-method/`:** Para agregar una nueva regla de negocio, editar solo `validarNegocio()` en `ProductoImportacionValidator`. Ver `ADR-001-template-method-importacion-masiva.md`.
