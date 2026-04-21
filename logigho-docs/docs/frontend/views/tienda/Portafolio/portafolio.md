---
autor: Adalberto González
fecha_creacion: 2026-04-20
ultima_actualizacion: 2026-04-20
estado: Desarrollo
nivel: 2
---

# Vista: Portafolio

**Autor:** Por definir  
**Selector:** `app-portafolio`  
**Ubicación:** `SitioLogiGho/src/app/views/tienda/portafolio`

---

## ¿Qué hace?

Catálogo de gestión de productos de la tienda. Permite al usuario listar, buscar, filtrar, crear, editar y eliminar productos. Muestra estadísticas de inventario en tiempo real.

---

## Ruta

| Propiedad | Valor |
|---|---|
| **Ruta** | `/tienda/portafolio` |
| **Título de página** | `Portafolio` |
| **Guard** | `AuthGuard` |
| **Rol requerido** | Todos (eliminación restringida a `CEO`) |
| **Parámetros de URL** | Ninguno |

### Definición en `routes.ts`

```typescript
{
  path: 'portafolio',
  loadComponent: () =>
    import('./portafolio/portafolio.component').then(m => m.PortafolioComponent),
  data: { title: 'Portafolio' }
}
```

---

## Estructura de archivos

```
portafolio/
├── portafolio.component.ts
├── portafolio.component.html
├── portafolio.component.scss
└── portafolio.component.spec.ts
```

---

## Secciones de la vista

| # | Sección | Descripción |
|---|---|---|
| 1 | **Cabecera** | Título, subtítulo y botones de acción: `Descargar inventario` y `Nuevo Producto` |
| 2 | **Estadísticas** | 4 tarjetas: Total productos, Disponibles, De proveedor y Dropshipping |
| 3 | **Barra de filtros** | Buscador de texto, dropdowns de tienda / categoría / proveedor y tabs por tipo |
| 4 | **Tabla de productos** | Listado paginado con imagen, nombre, ID, variación, precios, inventario, tipo, proveedor, estado y acciones |
| 5 | **Modal creación** | `app-creacion-productos` — formulario para crear uno o varios productos en lote |
| 6 | **Modal edición** | `app-actualizacion-productos` — formulario para editar un producto existente |

---

## Propiedades del componente

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `productos` | `Producto[]` | `[]` | Lista completa cargada desde el backend |
| `filteredProducts` | `Producto[]` | `[]` | Lista resultado de aplicar todos los filtros activos |
| `searchTerm` | `string` | `''` | Texto del buscador libre |
| `selectedTiendaFiltro` | `string` | `''` | Tienda seleccionada en el dropdown |
| `selectedCategoriaFiltro` | `string` | `''` | Categoría seleccionada en el dropdown |
| `selectedProveedorFiltro` | `string` | `''` | Proveedor seleccionado en el dropdown |
| `selectedFilter` | `string` | `''` | Tipo de producto activo en los tabs (`Propia`, `Proveedor`, `Dropshipping` o vacío para Todos) |
| `currentPage` | `number` | `1` | Página actual de la tabla |
| `pageSize` | `number` | `10` | Registros por página (opciones: 10, 25, 50, 100) |
| `isCreProdModalOpen` | `boolean` | `false` | Controla la visibilidad del modal de creación |
| `modalActualizarAbierto` | `boolean` | `false` | Controla la visibilidad del modal de edición |
| `productoParaEditar` | `Producto \| null` | `null` | Producto seleccionado para editar |
| `rolesUsuario` | `string[]` | `[]` | Roles del usuario activo leídos de `sessionStorage` |

---

## Interfaz `Producto`

```typescript
interface Producto {
  _id: string;         // ObjectId de MongoDB
  id: number;
  nombre: string;
  imagenUrl: string;
  imagenBase64?: string;
  precioventa: number;
  cantidad: number;
  idproducto: string;
  idvariacion: string;
  variacion: string;
  precioproveedor: number;
  sku: string;
  tipoproducto: string;
  IdTienda: string;
  Tienda: string;
  proveedor: string;
  categoria: string;
  peso: number;
  largo: number;
  ancho: number;
  alto: number;
  color: string;
  estado: string;
  perfilproducto: string;
}
```

---

## Flujo de inicialización

```
ngOnInit()
  → Lee roles_asignados de sessionStorage
  → fetchTableData()
      → Lee tiendas_asignadas de sessionStorage
      → GET metodoGenerico?coleccion=Productos&Tienda=...  (página 1)
      → decompressGzip(data.Resultado)
      → Mapea resultado a Producto[]
      → this.productos = resultado
      → this.filteredProducts = [...this.productos]
  → Construye dropdowns de tienda, categoría y proveedor
    extrayendo valores únicos de this.productos
```

---

## Métodos

### `fetchTableData()`
Carga los productos desde el backend filtrando por las tiendas asignadas al usuario en `sessionStorage`.

**Proceso:**
1. Lee `tiendas_asignadas` de `sessionStorage`
2. Llama a `metodoGenerico?coleccion=Productos&Tienda=...` con página `"1"`
3. Descomprime la respuesta con `DecompressionService`
4. Mapea los datos al modelo `Producto[]`
5. Inicializa `filteredProducts` con todos los productos

**Manejo de error:** Muestra `Swal.fire` con el mensaje del error si la llamada falla.

---

### `aplicarFiltros()`
Filtra `this.productos` combinando todos los criterios activos simultáneamente y actualiza `filteredProducts`. Resetea la paginación a página 1.

**Criterios acumulativos:**

| Criterio | Variable que lo controla |
|---|---|
| Texto libre | `searchTerm` (nombre, ID, variación, color, precios, proveedor) |
| Tienda | `selectedTiendaFiltro` |
| Categoría | `selectedCategoriaFiltro` |
| Proveedor | `selectedProveedorFiltro` |
| Tipo de producto | `selectedFilter` |

---

### `eliminarProducto(producto)`
Elimina un producto de la colección `Productos` en MongoDB.

**Proceso:**
1. Verifica que el usuario tenga rol `CEO` o `Desarrollador` (guarda `puedeEliminar`)
2. Muestra confirmación con `Swal.fire`
3. Si confirmado: llama a `eliminarGenerico` pasando el `_id` de MongoDB
4. Recarga tabla y reconstruye dropdowns
5. Limpia todos los filtros activos

---

### `onProductoCreado()`
Callback invocado cuando el hijo `app-creacion-productos` emite `refreshDataEvent` tras una creación exitosa.

**Proceso:**
1. Llama a `fetchTableData()` para recargar la tabla
2. Reconstruye los dropdowns de tienda, categoría y proveedor con los nuevos datos

---

### `descargarInventario()`
Exporta los productos actualmente visibles (`filteredProducts`) a un archivo `.xlsx` usando la librería `XLSX`.

---

### `descargarCatalogo()`
Genera un PDF con `jsPDF` que incluye imagen, nombre, precio, cantidad, tipo y proveedor de cada producto. Las imágenes se cargan desde el bucket S3 via `GetObjectService`.

---

### Paginación

| Getter | Descripción |
|---|---|
| `totalPaginas` | `Math.ceil(filteredProducts.length / pageSize)` |
| `productosPaginados` | Slice de `filteredProducts` para la página actual |
| `paginasVisibles` | Algoritmo smart: muestra máximo 7 páginas con `...` para gaps |

---

## Servicios utilizados

| Servicio | Métodos usados | Propósito |
|---|---|---|
| `ConsumoGenericoService` | `consultarGenerico()`, `eliminarGenerico()` | Consulta y eliminación de productos |
| `DecompressionService` | `decompressGzip()` | Descomprime las respuestas del backend |
| `GetObjectService` | `obtenerObjeto()` | Descarga imágenes desde S3 para el catálogo PDF |
| `TiendasService` | `tiendas$` | Suscripción al listado de tiendas |
| `UsuarioTiendasService` | — | Inyectado, pendiente de uso activo |

---

## Endpoints que consume

| Método | Ruta | Cuándo |
|---|---|---|
| `GET` | `metodoGenerico?coleccion=Productos&Tienda={tiendas}` | Al inicializar y tras crear/editar/eliminar |
| `DELETE` | `metodoGenerico?coleccion=Productos&_id={_id}` | Al eliminar un producto |

---

## Estados de la vista

| Estado | Descripción | Qué muestra |
|---|---|---|
| Cargando | `fetchTableData()` en curso | Tabla vacía (sin spinner explícito) |
| Con datos | Productos cargados | Tabla con paginación y estadísticas |
| Sin resultados de filtro | Filtros activos no coinciden | Fila `"No se encontraron productos..."` |
| Error de carga | Falla la llamada al backend | `Swal.fire` de error |

---

## Subcomponentes

| Componente | Selector | Cuándo se monta | Evento de retorno |
|---|---|---|---|
| `CreacionProductosComponent` | `app-creacion-productos` | Siempre (oculto con `*ngIf` interno) | `(closeEvent)` → `closeCreProdModal()` / `(refreshDataEvent)` → `onProductoCreado()` |
| `ActualizacionProductosComponent` | `app-actualizacion-productos` | Siempre (oculto con `[abierto]`) | `(cerrar)` → `cerrarModalActualizar()` / `(actualizado)` → `despuesDeActualizar()` |

---

## Control de roles

El getter `puedeEliminar` verifica que el usuario tenga al menos uno de estos roles (leídos de `sessionStorage → roles_asignados`):

```typescript
get puedeEliminar(): boolean {
  return this.rolesUsuario.includes('CEO') || this.rolesUsuario.includes('Desarrollador');
}
```

El botón **Eliminar** en la tabla solo se renderiza si `puedeEliminar === true`.

---

## Changelog de la vista

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-04-20 | Por definir | Corrección race condition en generación de IDs al crear productos en lote |
| 2026-04-20 | Por definir | `fetchTableData()` refrescada al inicio de cada submit para reducir ventana de colisión |
| 2026-04-20 | Por definir | Adición de `onProductoCreado()` para refrescar tabla y dropdowns tras creación exitosa |
| 2026-04-20 | Por definir | `.trim()` en nombre de tienda al crear productos para evitar inconsistencias de filtro |

---

## Observaciones

- **Paginación desde el servicio:** El `fetchTableData()` del portafolio solo carga la página `"1"`. La API soporta hasta 3 000 registros antes de paginar, por lo que con el volumen actual no hay pérdida de datos. Si el catálogo supera ese umbral será necesario implementar iteración de páginas (ver patrón en `creacion-productos.component.ts → fetchTableData()`).
- **Inconsistencia de conteo:** Se detectó que algunos productos tienen el campo `Tienda` con espacios extra (`"Nombre tienda "`) que el filtro exacto del backend no retorna. El fix en frontend (`.trim()` al crear) previene nuevos casos pero los registros existentes en DB permanecen afectados hasta que se ejecute una migración en el backend.
- **Inconsistencia en la generacion de los IDs:** Se detecto que en la generación de IDs duplicados en varios productos, debido a que el Front es que calcula el ID y puede llegar tener problemas de Race contdition.

