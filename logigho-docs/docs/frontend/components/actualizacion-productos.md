---
autor: Adalberto González
fecha_creacion: 2026-04-23
ultima_actualizacion: 2026-04-23
estado: Desarrollo
nivel: 2
---

# Componente: ActualizacionProductos

**Autor:** Adalberto González  

**Selector:** `app-actualizacion-productos`  

**Ubicación:** `SitioLogiGho/src/app/components/actualizacion-productos`

---

## ¿Qué hace?

Modal de edición de un producto existente. Recibe el producto a editar desde el componente padre (`PortafolioComponent`), pre-carga todos sus campos en un formulario reactivo y permite modificar características, dimensiones, variación, etiquetas (tipo, proveedor, tienda) y estado. Al confirmar, persiste los cambios en la base de datos usando el `_id` de MongoDB. La imagen es de solo lectura en la vista actual.

---

## Diferencia clave respecto a `CreacionProductos`

A diferencia de `CreacionProductos`, este componente **sí usa `*ngIf="abierto"`**, por lo que se destruye completamente al cerrarse. Esto significa que no hay estado residual entre aperturas: el formulario y los dropdowns siempre parten limpios.

---

## Roles y acceso

| Acceso | Descripción |
|---|---|
| Autenticado | Requiere sesión activa. La apertura del modal la controla `PortafolioComponent`, que está protegido por `AuthGuard`. |

---

## Estructura de archivos

```
actualizacion-productos/
├── actualizacion-productos.component.ts
├── actualizacion-productos.component.html
└── actualizacion-productos.component.scss
```

---

## Inputs y Outputs

| Decorador | Nombre | Tipo | Descripción |
|---|---|---|---|
| `@Input` | `producto` | `Producto \| null` | El producto a editar. El padre lo inyecta al abrir el modal. |
| `@Input` | `abierto` | `boolean` | Controla si el modal es visible (vía `*ngIf`). |
| `@Output` | `cerrar` | `EventEmitter<void>` | Emitido al llamar a `close()`. El padre lo escucha para ocultar el modal. |
| `@Output` | `actualizado` | `EventEmitter<void>` | Emitido tras una actualización exitosa. El padre lo escucha en `despuesDeActualizar()` para recargar la tabla. |

---

## Interfaz `Producto` (interna)

```typescript
interface Producto {
  _id?: { $oid: string } | string;
  id?: number | null;
  nombre?: string;
  imagenUrl?: string;
  precioventa?: number | null;
  cantidad?: number | null;
  idproducto?: number | null | string;
  idvariacion?: string;
  variacion?: string;
  precioproveedor?: number | null;
  sku?: string;
  tipoproducto?: string;
  IdTienda?: string;
  Tienda?: string;
  proveedor?: string;
  categoria?: string;
  peso?: number | null;
  largo?: number | null;
  ancho?: number | null;
  alto?: number | null;
  color?: string;
  estado?: string;
  perfilproducto?: string;
}
```

Todos los campos son opcionales (`?`) a diferencia de la interfaz de creación, ya que el producto puede venir con datos parciales desde la tabla.

---

## Propiedades del componente

### Formulario

| Propiedad | Tipo | Descripción |
|---|---|---|
| `form` | `FormGroup` | Formulario reactivo con todos los campos del producto. |

### Listas de datos

| Propiedad | Tipo | Descripción |
|---|---|---|
| `tiendas` | `any[]` | Tiendas activas cargadas desde `Tienda` en la BD (filtradas por `Integracion=Interrapidisimo` y `NombreTienda`). |
| `categorias` | `any[]` | Categorías cargadas desde `ProductosCategorias`. |
| `proveedores` | `any[]` | Subconjunto de tiendas cuyo `TipoTienda` incluye `"Proveedor"`. |

### Estado de imagen

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `selectedFile` | `File \| null` | `null` | Archivo de imagen seleccionado. Declarado pero sin uso activo (la imagen no es editable en la vista actual). |
| `imagenPreviaUrl` | `string \| null` | `null` | URL de la imagen actual del producto para previsualización. |

### Estado de dropdowns

| Propiedad | Tipo | Descripción |
|---|---|---|
| `buscarCategoria` | `string` | Texto en el buscador de categoría. |
| `mostrarCategoriaMenu` | `boolean` | Visibilidad del dropdown de categoría. |
| `categoriasFiltradas` | `any[]` | Lista filtrada de categorías. |
| `categoriaSeleccionadaNombre` | `string` | Nombre de la categoría seleccionada (mostrado en el placeholder). |
| `buscarProveedor` | `string` | Texto en el buscador de proveedor. |
| `mostrarProveedorMenu` | `boolean` | Visibilidad del dropdown de proveedor. |
| `proveedoresFiltrados` | `any[]` | Lista filtrada de proveedores. |
| `proveedorSeleccionadoNombre` | `string` | Nombre del proveedor seleccionado. |
| `buscarTienda` | `string` | Texto en el buscador de tienda. |
| `mostrarTiendaMenu` | `boolean` | Visibilidad del dropdown de tienda. |
| `tiendasFiltradas` | `any[]` | Lista filtrada de tiendas. |
| `tiendaSeleccionadaNombre` | `string` | Nombre de la tienda seleccionada. |
| `buscarTipoProducto` | `string` | Texto en el buscador de tipo de producto. |
| `mostrarTipoProductoMenu` | `boolean` | Visibilidad del dropdown de tipo de producto. |
| `tipoProductoFiltradas` | `any[]` | Lista filtrada de tipos de producto. |
| `tipoProductoSeleccionadoNombre` | `string` | Nombre del tipo seleccionado. |
| `tipoProductoOpciones` | `{ Nombre, Value }[]` | Lista estática: `Dropshipping`, `Propia`, `Proveedor`. |

---

## Secciones de la vista (modal)

| # | Sección | Descripción |
|---|---|---|
| 1 | **Cabecera** | Título "Actualización de Producto" y botón de cierre (`×`). |
| 2 | **Características** | Nombre, precio de venta, precio proveedor, cantidad y categoría (con dropdown buscador). |
| 3 | **Imagen del Producto** | Muestra la imagen actual como previsualización. No es editable desde la UI. |
| 4 | **Dimensiones** | Ancho, alto, largo y peso en sus respectivas unidades. |
| 5 | **Características (variación)** | Variación, color y SKU del producto. |
| 6 | **Etiquetas del producto** | Tipo de producto, proveedor y tienda mediante buscadores con dropdown custom. |
| 7 | **Otros** | Estado (`Activo`/`Inactivo`) y perfil del producto (`Propio`/`Publico`/`Privado`). |
| 8 | **Footer** | Botones "Cerrar" (`close()`) y "Actualizar" (`onActualizar()`). |

---

## Flujo de inicialización

```
ngOnInit()
  → fetchTableDataTienda()        (carga tiendas activas desde BD)
  → fetchTableDataCategoria()     (carga categorías desde BD)
  → fetchTableDataProveedor()     (carga proveedores desde BD)

ngOnChanges()
  → si producto cambia y abierto = true:
      → cargarDatosProducto()     (pre-carga el formulario con los datos del producto)
  → si abierto cambia a false:
      → resetForm()               (limpia el formulario y todos los estados de dropdown)
```

---

## Métodos

### `cargarDatosProducto()`
Pre-carga el formulario con los datos del `@Input() producto`.

**Proceso:**
1. Busca en las listas locales (`tiendas`, `proveedores`, `categorias`) el ítem cuyo `Nombre` o `Id` coincida con los campos del producto.
2. Hace `patchValue` en el formulario con todos los campos del producto, asignando los IDs encontrados a `nombreTienda`, `nombreProveedor` y `nombreCategoria`.
3. Establece los nombres en los placeholders de los dropdowns (`tiendaSeleccionadaNombre`, `proveedorSeleccionadoNombre`, `categoriaSeleccionadaNombre`, `tipoProductoSeleccionadoNombre`).
4. Asigna `imagenPreviaUrl` para la previsualización.

---

### `resetForm()`
Limpia el formulario y el estado de todos los dropdowns al cerrar el modal.

**Qué limpia:**
- `form.reset()`
- `selectedFile` → `null`
- `imagenPreviaUrl` → `null`
- Nombres seleccionados de los cuatro dropdowns
- Campos de búsqueda (`buscarCategoria`, `buscarProveedor`, `buscarTienda`, `buscarTipoProducto`)
- Input de archivo (si existe el `@ViewChild`)

---

### `onActualizar(): Promise<void>`
Método principal de actualización.

**Proceso:**
1. Valida que `producto._id` exista; si no, muestra `Swal.fire` de error.
2. Muestra `Swal.fire` de carga con spinner.
3. Si `selectedFile` tiene valor, sube la imagen con `cargarImagenProducto()` y actualiza `imagenUrl` (no ocurre en la UI actual porque no hay input de archivo).
4. Resuelve nombres de tienda, proveedor y categoría buscando en sus listas locales.
5. Construye `productoActualizado` haciendo spread de `this.producto` y sobreescribiendo con los valores del formulario.
6. Extrae el `_id` real de MongoDB (maneja tanto `string` como `{ $oid: string }`).
7. Llama a `actualizarGenerico(productoActualizado, productoId, 'metodoGenerico?coleccion=Productos')`.
8. Emite `actualizado` y llama a `close()` al finalizar.

**Manejo de error:** Cualquier excepción muestra `Swal.fire` de error con el mensaje de la excepción.

---

### `fetchTableDataTienda(): Promise<void>`
Carga tiendas activas filtrando por `Estado=ACTIVO`, `Integracion=Interrapidisimo` y `NombreTienda` (tiendas asignadas al usuario). Descomprime con `decompressZstd`. Ordena alfabéticamente y mapea a `{ Nombre, Id }`.

**Diferencia respecto a `CreacionProductos`:** Incluye los filtros adicionales `Integracion` y `NombreTienda` en la query, lo que restringe el resultado a tiendas con integración activa asignadas al usuario.

---

### `fetchTableDataCategoria(): Promise<void>`
Carga categorías desde `ProductosCategorias`. Descomprime con `decompressGzip`. Mapea a `{ Nombre, Id }`.

---

### `fetchTableDataProveedor(): Promise<void>`
Carga tiendas con `TipoTienda` que contiene `"Proveedor"`, usando los mismos filtros adicionales que `fetchTableDataTienda`. Descomprime con `decompressZstd`. Mapea a `{ Nombre, Id, TipoTienda }`.

---

### `cargarImagenProducto(key, userData): Promise<void>`
Sube una imagen al bucket S3 `imagenes-productos-logigho`.

**Nota:** Declarado e implementado, pero no se invoca desde la vista actual porque el campo de imagen no es editable. Está disponible para cuando se habilite la edición de imágenes.

---

### Métodos de dropdown (patrón repetido para categoría, proveedor, tienda y tipo de producto)

A diferencia de `CreacionProductos`, los dropdowns aquí **no están indexados por array** — son variables simples (no por formulario), ya que solo existe un producto a la vez.

| Método | Descripción |
|---|---|
| `filtrarXxx()` | Filtra la lista completa según el texto en `buscarXxx` (case-insensitive). |
| `onXxxSeleccionado(item)` | Guarda el nombre en `xxxSeleccionadoNombre`, limpia la búsqueda, cierra el menú y hace `patchValue` con el ID del ítem. |
| `abrirXxxMenu()` | Cierra todos los menús, abre el de `xxx` y puebla la lista sin filtro. |
| `alternadorXxxMenu()` | Alterna la visibilidad del menú de `xxx`. Si se abre, puebla la lista sin filtro. |
| `cerrarTodosLosMenus()` | Escuchador `@HostListener('document:click')`. Pone todos los `mostrarXxxMenu` en `false`. |

---

## Servicios Angular utilizados

| Servicio | Métodos usados | Propósito |
|---|---|---|
| `ConsumoGenericoService` | `consultarGenerico()`, `actualizarGenerico()` | Consulta de tiendas, categorías y proveedores; actualización del producto |
| `DecompressionService` | `decompressGzip()`, `decompressZstd()` | Descomprime las respuestas del backend |
| `PutObjectService` | `cargarInformacion()` | Sube imágenes a S3 (disponible pero sin uso activo en la UI actual) |
| `GetObjectService` | — | Inyectado pero sin uso activo en este componente |
| `TiendasService` | — | Inyectado pero sin uso activo en este componente |
| `FormBuilder` | `group()` | Construcción del formulario reactivo |

---

## Dependencias externas

| Librería | Uso |
|---|---|
| `sweetalert2` | Alertas de carga, éxito y error durante la actualización |

---

## Endpoints que consume

| Método | Ruta | Cuándo |
|---|---|---|
| `GET` | `metodoGenerico?coleccion=Tienda&Estado=ACTIVO&Integracion=Interrapidisimo&NombreTienda=...` | Al inicializar (tiendas y proveedores) |
| `GET` | `metodoGenerico?coleccion=ProductosCategorias` | Al inicializar (categorías) |
| `PUT/PATCH` | `metodoGenerico?coleccion=Productos` (vía `actualizarGenerico`) | Al confirmar la actualización |

---

## Changelog del componente

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-04-23 | Adalberto González | Creación del componente de actualización de productos con formulario reactivo |
| 2026-04-23 | Adalberto González | Migración de `<select>` nativos a buscadores con dropdown custom para categoría, proveedor, tienda y tipo de producto |

---

## Observaciones

- **Imagen no editable:** El HTML muestra la imagen como previsualización con el mensaje "La imagen del producto no es editable", pero el TypeScript tiene `onFileSelected`, `selectedFile` y `cargarImagenProducto` preparados. La edición de imagen puede habilitarse añadiendo un `<input type="file">` al template sin cambios en el `.ts`.
- **`GetObjectService` y `TiendasService` sin uso:** En el codigo pero sin llamadas activas.
- **Filtro de tiendas diferente al de creación:** `fetchTableDataTienda` y `fetchTableDataProveedor` incluyen `Integracion=Interrapidisimo` y `NombreTienda`, lo que puede dar un resultado distinto al de `CreacionProductos`. Conviene unificar el criterio de consulta.
- **`cargarDatosProducto` depende de que las listas estén cargadas:** Si `ngOnChanges` dispara antes de que `fetchTableDataTienda/Categoria/Proveedor` terminen, los dropdowns no mostrarán el ítem preseleccionado correctamente.
- **Sin validadores en el formulario:** Al igual que en creación, no hay `Validators` definidos, por lo que el botón "Actualizar" no bloquea envíos con campos vacíos.
- **Ciclo de vida limpio:** Al usar `*ngIf="abierto"`, el componente se destruye al cerrar. No hay problemas de estado residual entre aperturas, a diferencia de `CreacionProductos`.
