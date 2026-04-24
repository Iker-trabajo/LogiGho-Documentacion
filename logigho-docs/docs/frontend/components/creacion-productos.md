---
autor: Adalberto González
fecha_creacion: 2026-04-23
ultima_actualizacion: 2026-04-23
estado: Desarrollo
nivel: 2
---

# Componente: CreacionProductos

**Autor:** Adalberto González  

**Selector:** `app-creacion-productos`  

**Ubicación:** `SitioLogiGho/src/app/components/creacion-productos`

---

## ¿Qué hace?

Modal de creación masiva de productos. Permite al usuario rellenar uno o varios formularios en paralelo, seleccionar tienda, categoría, proveedor y tipo de producto mediante buscadores con dropdown, subir una imagen a S3 y guardar cada producto en la base de datos con un ID, SKU e ID de variación generados de forma consecutiva.

---

## Roles y acceso

| Acceso | Descripción |
|---|---|
| Autenticado | Requiere sesión activa. La apertura del modal la controla el componente padre (`PortafolioComponent`), que a su vez está protegido por `AuthGuard`. |

---

## Estructura de archivos

```
creacion-productos/
├── creacion-productos.component.ts
├── creacion-productos.component.html
└── creacion-productos.component.scss
```

---

## Inputs y Outputs

| Decorador | Nombre | Tipo | Descripción |
|---|---|---|---|
| `@Input` | `isOpen` | `boolean` | Controla si el modal es visible. El padre lo pone en `true` al abrir y en `false` al cerrar. |
| `@Input` | `transactionId` | `string` | ID de transacción inyectado por el padre (actualmente recibido pero no utilizado internamente). |
| `@Output` | `closeEvent` | `EventEmitter<void>` | Emitido al llamar a `close()`. El padre lo escucha para ocultar el modal. |
| `@Output` | `refreshDataEvent` | `EventEmitter<void>` | Emitido al terminar una creación exitosa. El padre lo escucha en `onProductoCreado()` para recargar la tabla. |

---

## Interfaz `Producto` (interna)

```typescript
interface Producto {
  _id?: { $oid: string } | string;
  id: number | null;
  nombre: string;
  imagenUrl: string;
  precioventa: number | null;
  cantidad: number | null;
  idproducto: number | null;
  idvariacion: string;
  variacion: string;
  precioproveedor: number | null;
  sku: string;
  tipoproducto: string;
  IdTienda?: string;
  Tienda?: string;
  proveedor: string;
  categoria: string;
  peso: number | null;
  largo: number | null;
  ancho: number | null;
  alto: number | null;
  color: string;
  estado: string;
  perfilproducto: string;
  FechaCreacionProducto?: string;
}
```

---

## Propiedades del componente

### Formulario

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `formGroupWrapper` | `FormGroup` | — | Formulario raíz que envuelve el `FormArray`. |
| `formArray` | `FormArray` | `[]` | Array de `FormGroup`, uno por cada producto a crear en lote. |
| `nuevoProducto` | `Producto` | objeto vacío | Objeto auxiliar donde se acumula la tienda, proveedor, categoría e imagen antes de construir `productoParaGuardar`. |

### Listas de datos

| Propiedad | Tipo | Descripción |
|---|---|---|
| `tiendas` | `any[]` | Tiendas activas cargadas desde `Tienda` en la BD. |
| `categorias` | `any[]` | Categorías cargadas desde `ProductosCategorias`. |
| `proveedores` | `any[]` | Subconjunto de tiendas cuyo `TipoTienda` incluye `"Proveedor"`. |
| `tipoProductoOpciones` | `{ Nombre, Value }[]` | Lista estática: `Dropshipping`, `Propia`, `Proveedor`. |

### Estado general

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `isLoading` | `boolean` | `false` | Bloquea el botón "Guardar" mientras se procesa la creación. |
| `imagenesSeleccionadas` | `(File \| null)[]` | `[]` | Imagen seleccionada por formulario (indexada por posición en `formArray`). |
| `consecutivoVariacion` | `number` | `1` | Contador que se incrementa al generar cada `idvariacion` dentro de un lote. |
| `rows` | `TablaRow[]` | `[]` | Datos de todos los productos cargados (usado solo para calcular IDs consecutivos). |
| `rowsMemory` | `any[]` | `[]` | Buffer intermedio del procesamiento por bloques de `fetchTableData()`. |

### Estado de dropdowns (por formulario)

Cada uno de los siguientes es un array indexado por posición en `formArray`:

| Propiedad | Tipo | Descripción |
|---|---|---|
| `buscarCategorias` | `String[]` | Texto escrito en el buscador de categoría. |
| `mostrarCategoriasMenu` | `boolean[]` | Visibilidad del dropdown de categoría. |
| `categoriasFiltradas` | `any[][]` | Lista filtrada de categorías para mostrar. |
| `categoriaSeleccionadaNombre` | `string[]` | Nombre de la categoría seleccionada (mostrado en el placeholder). |
| `buscarProveedores` | `String[]` | Texto escrito en el buscador de proveedor. |
| `mostrarProveedoresMenu` | `boolean[]` | Visibilidad del dropdown de proveedor. |
| `proveedoresFiltrados` | `any[][]` | Lista filtrada de proveedores. |
| `proveedorSeleccionadoNombre` | `String[]` | Nombre del proveedor seleccionado. |
| `buscarTiendas` | `String[]` | Texto escrito en el buscador de tienda. |
| `mostrarTiendasMenu` | `boolean[]` | Visibilidad del dropdown de tienda. |
| `tiendasFiltradas` | `any[][]` | Lista filtrada de tiendas. |
| `tiendaSeleccionadaNombre` | `String[]` | Nombre de la tienda seleccionada. |
| `buscarTipoProducto` | `String[]` | Texto escrito en el buscador de tipo. |
| `mostrarTipoProductoMenu` | `boolean[]` | Visibilidad del dropdown de tipo. |
| `tipoProductoFiltradas` | `any[][]` | Lista filtrada de tipos de producto. |
| `tipoProductoSeleccionadoNombre` | `String[]` | Nombre del tipo seleccionado. |

---

## Secciones de la vista (modal)

| # | Sección | Descripción |
|---|---|---|
| 1 | **Cabecera** | Título "Creación de Productos" y botón de cierre (`×`). |
| 2 | **Formulario de producto** (`*ngFor` sobre `formArray`) | Para cada producto: campos de características, imagen, dimensiones, variación/color, etiquetas (tipo, proveedor, tienda) y otros (estado, perfil). |
| 3 | **Buscadores con dropdown** | Categoría, tipo de producto, proveedor y tienda usan un `input` de búsqueda con dropdown custom en lugar de `<select>`. |
| 4 | **Botón "Añadir Producto"** | Llama a `addFormularioProducto()` para agregar otro bloque de formulario. |
| 5 | **Footer** | Botones "Cerrar" (`close()`) y "Guardar" (`onCreateOrModify()`). El botón "Guardar" se deshabilita mientras `isLoading` es `true`. |

---

## Flujo de inicialización

```
ngOnInit()
  → addFormularioProducto()       (agrega el primer FormGroup)
  → fetchTableDataTienda()        (carga tiendas activas desde BD)
  → fetchTableDataCategoria()     (carga categorías desde BD)
  → fetchTableDataProveedor()     (carga proveedores desde BD)
  → tiendaService.tiendas$.subscribe()  (suscripción reactiva, sobreescribe this.tiendas)

ngOnChanges()
  → si isOpen cambia a true:
      → fetchTableData()          (carga todos los productos para cálculo de IDs)
```

---

## Métodos

### `addFormularioProducto()`
Agrega un nuevo bloque de formulario al lote.

**Proceso:**
1. Obtiene el índice actual (`formArray.length`)
2. Inicializa las posiciones `[index]` de todos los arrays de estado de dropdown en sus valores vacíos/falsos
3. Llama a `createProductoForm()` y hace `push` al `formArray`

---

### `removeFormularioProducto(index)`
Elimina el formulario en la posición `index` del lote.

**Proceso:**
1. `formArray.removeAt(index)`
2. Hace `splice(index, 1)` en todos los arrays de estado de dropdown para mantener sincronía con `formArray`

---

### `createProductoForm(): FormGroup`
Retorna un `FormGroup` nuevo con todos los controles del formulario de un producto (`nombre`, `imagenUrl`, `precioventa`, `cantidad`, `precioproveedor`, `sku`, `tipoproducto`, `Tienda`, `IdTienda`, `proveedor`, `categoria`, `nombreTienda`, `nombreProveedor`, `nombreCategoria`, `peso`, `largo`, `ancho`, `alto`, `color`, `estado`, `perfilproducto`).

---

### `onCreateOrModify(): Promise<void>`
Método principal de guardado. Crea todos los productos del lote secuencialmente.

**Proceso:**
1. Bloquea re-envíos con `isLoading`
2. Muestra `Swal.fire` de carga con spinner
3. Llama a `fetchTableData()` para obtener IDs actualizados (mitiga race condition)
4. Calcula `idProductoGenerado` y `idGenerado` base con `obtenerIdProductoConsecutivo()` / `obtenerIdConsecutivo()`
5. Itera sobre cada `FormGroup` del `formArray`:
   - Genera el SKU base: `NOM-variacion-COL`
   - Verifica unicidad del SKU con `verificarExistenciaSKU()`; si existe, agrega consecutivo (`-1`, `-2`, …)
   - Valida que tienda, categoría y proveedor estén seleccionados
   - Valida que haya una imagen seleccionada
   - Sube la imagen con `cargarImagenProducto()`
   - Resuelve nombres de tienda, proveedor y categoría buscando en sus listas locales
   - Construye `productoParaGuardar` con todos los campos
   - Llama a `insertarGenerico(productoParaGuardar, 'productos')`
   - Llama a `insertarGenerico(bodyActualizaInventario, 'actualizaInventarioLotes')` para actualizar stock
   - Incrementa `idProductoGenerado`
6. Emite `refreshDataEvent` y llama a `close()` al finalizar el lote
7. En `finally`: libera `isLoading`

**Manejo de error:** Cualquier excepción muestra `Swal.fire` de error.

---

### `fetchTableData(): Promise<void>` *(privado)*
Carga la totalidad de productos de la BD para calcular IDs consecutivos correctos.

**Proceso:**
1. Consulta página 1 para obtener `NumeroPaginas`
2. Crea grupos de 8 páginas y las procesa en paralelo con `Promise.all`
3. Descomprime cada página con `decompressGzip`
4. Procesa los datos en bloques de 128 items con `processBlockAsync()` (usa `requestIdleCallback` si disponible)
5. Genera columnas con `generateColumns()` y asigna `this.rows`

**Nota:** Este método es costoso (carga todos los productos). Se llama al abrir el modal (`ngOnChanges`) y al inicio de `onCreateOrModify()` para reducir la ventana de race condition en la generación de IDs.

---

### `fetchTableDataTienda(): Promise<void>`
Carga las tiendas activas desde la colección `Tienda` (filtradas por `Estado=ACTIVO`). Descomprime con `decompressZstd`. Ordena alfabéticamente y mapea a `{ Nombre, Id }`.

---

### `fetchTableDataCategoria(): Promise<void>`
Carga categorías desde `ProductosCategorias`. Descomprime con `decompressGzip`. Mapea a `{ Nombre, Id }`.

---

### `fetchTableDataProveedor(): Promise<void>`
Carga tiendas con `TipoTienda` que contiene `"Proveedor"` desde la colección `Tienda`. Descomprime con `decompressZstd`. Mapea a `{ Nombre, Id, TipoTienda }`.

---

### `verificarExistenciaSKU(sku): Promise<boolean>`
Consulta `metodoGenerico?coleccion=Productos&sku={sku}`. Retorna `true` si hay al menos un resultado, `false` si no. Usado dentro del loop de `onCreateOrModify()` para garantizar unicidad de SKU.

---

### `onFileSelected(event, index)`
Valida que el archivo sea `image/jpeg`, `image/png` o `image/webp`. Si el formato no es válido muestra `Swal` de error y limpia el input. Si es válido, guarda el `File` en `imagenesSeleccionadas[index]`.

---

### `imagenPrevia(file): string`
Retorna `URL.createObjectURL(file)` para la previsualización en el template. Retorna `''` si `file` es `null`.

---

### `cargarImagenProducto(key, userData): Promise<void>`
Sube una imagen al bucket S3 `imagenes-productos-logigho`.

**Proceso:**
1. Convierte el archivo a base64 con `convertirABase64()`
2. Extrae la parte base64 pura (sin prefijo MIME)
3. Llama a `putObjectService.cargarInformacion()` con el base64, bucket, clave y extensión
4. Asigna la URL CloudFront a `nuevoProducto.imagenUrl`

---

### `obtenerIdConsecutivo(): number`
Retorna `Math.max(...rows.map(p => p.id)) + 1`. Retorna `1` si no hay productos.

---

### `obtenerIdProductoConsecutivo(): number`
Retorna `Math.max(...rows.map(p => p.idproducto)) + 1`. Retorna `10001` si no hay productos.

---

### `obtenerIdConsecutivoVariacion(): string`
Retorna el valor actual de `consecutivoVariacion` como string y lo incrementa. Se llama una vez por producto dentro del lote para generar IDs de variación únicos dentro de esa sesión.

---

### `generarFechaCreacion(): string`
Retorna la fecha y hora actual ajustada a UTC-5 en formato `"YYYY-MM-DD HH:mm:ss"`.

---

### Métodos de dropdown (patrón repetido para categoría, proveedor, tienda y tipo de producto)

| Método | Descripción |
|---|---|
| `filtrarXxx(formIndex)` | Filtra la lista completa según el texto en `buscarXxx[formIndex]` (case-insensitive) y actualiza `xxxFiltrados[formIndex]`. |
| `xxxSeleccionado(item, formIndex)` | Guarda el nombre del ítem en `xxxSeleccionadoNombre[formIndex]`, limpia el campo de búsqueda, cierra el menú y hace `patchValue` en el `FormControl` correspondiente del `FormGroup` (guarda el ID del ítem, no el nombre). |
| `abrirXxxMenu(formIndex)` | Cierra todos los menús, abre el de `xxx` en `formIndex` y puebla la lista sin filtro. |
| `alternadorXxxMenu(formIndex)` | Alterna la visibilidad del menú de `xxx` en `formIndex`. Si se abre, puebla la lista sin filtro. |
| `cerrarTodosLosMenus()` | Escuchador `@HostListener('document:click')`. Pone todos los `mostrarXxxMenu` en `false`. |

---

## Servicios Angular utilizados

| Servicio | Métodos usados | Propósito |
|---|---|---|
| `ConsumoGenericoService` | `consultarGenerico()`, `insertarGenerico()` | Consulta de productos, tiendas, categorías y proveedores; inserción de productos e inventario |
| `DecompressionService` | `decompressGzip()`, `decompressZstd()` | Descomprime las respuestas del backend |
| `PutObjectService` | `cargarInformacion()` | Sube imágenes a S3 |
| `GetObjectService` | — | Inyectado pero sin uso activo en este componente |
| `TiendasService` | `tiendas$` | Suscripción reactiva al listado de tiendas (sobreescribe el resultado de `fetchTableDataTienda`) |
| `FormBuilder` | `group()`, `array()` | Construcción de formularios reactivos |

---

## Dependencias externas

| Librería | Uso |
|---|---|
| `sweetalert2` | Alertas de carga, éxito y error durante la creación |
| `rxjs/firstValueFrom` | Convierte observables en promesas dentro de los métodos `async` |
| `xlsx` | Importado pero sin uso activo en este componente |

---

## Endpoints que consume

| Método | Ruta | Cuándo |
|---|---|---|
| `GET` | `metodoGenerico?coleccion=Productos` | Al abrir el modal y al iniciar `onCreateOrModify()` (todas las páginas) |
| `GET` | `metodoGenerico?coleccion=Tienda&Estado=ACTIVO` | Al inicializar (tiendas y proveedores) |
| `GET` | `metodoGenerico?coleccion=ProductosCategorias` | Al inicializar (categorías) |
| `GET` | `metodoGenerico?coleccion=Productos&sku={sku}` | Por cada producto, para verificar unicidad de SKU |
| `POST` | `productos` (vía `insertarGenerico`) | Por cada producto aprobado en el lote |
| `POST` | `actualizaInventarioLotes` (vía `insertarGenerico`) | Inmediatamente tras insertar cada producto |

---

## Changelog del componente

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-04-20 | Por definir | Migración de `<select>` nativos a buscadores con dropdown custom para categoría, proveedor, tienda y tipo de producto |
| 2026-04-20 | Por definir | Separación del estado de dropdown en arrays indexados para soportar creación en lote (múltiples formularios) |
| 2026-04-20 | Por definir | Adición de `fetchTableData()` al inicio de `onCreateOrModify()` para reducir ventana de race condition en generación de IDs |
| 2026-04-20 | Por definir | `.trim()` sobre `tiendaSeleccionada.Nombre` al asignar `nuevoProducto.Tienda` para evitar espacios que rompan el filtro de portafolio |
| 2026-04-20 | Por definir | Adición de llamada a `actualizaInventarioLotes` tras cada inserción para registrar ingreso de inventario |

---

## Observaciones

- **Race condition en IDs:** Se generan en frontend, riesgo de duplicados. Debe hacerlo el backend.
- **`consecutivoVariacion` no se reinicia:** Sigue aumentando si no se destruye el componente.
- **Conflicto en `this.tiendas`:** Dos formatos distintos causan errores.
- **`loadFile()` incompleto:** No descarga nada.
- **`transactionId` sin uso:** No se utiliza.
- **Validación débil:** Sin `Validators`, permite envíos inválidos.