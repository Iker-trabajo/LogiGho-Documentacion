---

## Autor: Adalberto González

Fecha creacion: 2026-05-20  
Estado: desarrollo  
Tipo: componente

# Componente: CreacionesMasivaProductos

**Selector:** `app-creaciones-masiva-productos`  
**Ubicación:** `src/app/components/creaciones-masiva-productos`  
**Acceso:** Autenticado | Rol: Todos

---

## ¿Qué hace?

Modal de 3 pasos (stepper) que permite cargar un archivo Excel con múltiples productos y crearlos en lote en la colección `Productos` de MongoDB. Valida cada fila con reglas estructurales y de negocio antes de permitir la importación, gestiona la asignación de imágenes por producto, y muestra un resumen de creados vs. fallidos al finalizar.

---

## Decoradores

| Decorador | Nombre | Tipo | Descripción |
| --------- | ------ | ---- | ----------- |
| `@Input` | `isOpen` | `boolean` | Controla la visibilidad del modal. Lo gestiona `PortafolioComponent`. |
| `@Output` | `closeEvent` | `EventEmitter<void>` | Emite cuando el usuario cierra el modal para que el padre actualice su flag de apertura. |
| `@Output` | `refreshDataEvent` | `EventEmitter<void>` | Emite cuando al menos un producto fue creado, para que el padre recargue la tabla. |
| `@ViewChild` | `inputArchivo` | `ElementRef<HTMLInputElement>` | Referencia al input de archivo para limpiarlo programáticamente en `limpiarArchivo()`. |

---

## Propiedades clave

| Propiedad | Tipo | Descripción |
| --------- | ---- | ----------- |
| `paso` | `1 \| 2 \| 3` | Paso activo del stepper. |
| `archivoSeleccionado` | `File \| null` | Archivo Excel cargado por el usuario en el paso 1. |
| `filasPreview` | `FilaPreview[]` | Resultado de evaluar todas las filas del Excel (válidas e inválidas). |
| `filasValidas` | `FilaPreview[]` | Copia de las filas que pasaron todas las validaciones. Se usa en pasos 2 y 3. |
| `productoActualIndex` | `number` | Índice del producto mostrado en el paso 2 de asignación de imágenes. |
| `resumen` | `ResumenItem[]` | Resultado de la importación: uno por cada fila válida procesada. |
| `importando` | `boolean` | `true` mientras se ejecuta la importación en el paso 3. Bloquea el botón de cerrar. |
| `validator` | `ProductoImportacionValidator` | Instancia del validador concreto (Template Method). |

---

## Servicios y endpoints

| Servicio | Método | Endpoint | Cuándo |
| -------- | ------ | -------- | ------ |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET metodoGenerico?coleccion=Tienda&Estado=ACTIVO` | Al abrir el modal (tiendas y proveedores) |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET metodoGenerico?coleccion=ProductosCategorias` | Al abrir el modal (categorías) |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET metodoGenerico?coleccion=Productos` | Al abrir el modal y antes de crear cada producto (race condition) |
| `ConsumoGenericoService` | `insertarGenerico()` | `POST productos` | Por cada producto válido en el paso 3 |
| `ConsumoGenericoService` | `insertarGenerico()` | `POST actualizaInventarioLotes` | Por cada producto creado (registro de ingreso de inventario) |
| `GetObjectService` | `obtenerObjeto()` | S3 `logigho-plantillas/Productos.xlsx` | Al abrir el modal (descarga la plantilla) |
| `PutObjectService` | `cargarInformacion()` | S3 `imagenes-productos-logigho` | Por cada producto con imagen asignada en el paso 2 |

---

## Flujo principal

```
ngOnChanges() [isOpen → true]
  → resetear()               — limpia estado de sesión anterior
  → cargarDatosBD()          — en paralelo:
      → cargarTiendas()      — GET Tienda (Zstd)
      → cargarCategorias()   — GET ProductosCategorias (Gzip)
      → cargarProveedores()  — GET Tienda filtrado por TipoTienda=Proveedor (Zstd)
      → cargarProductosExistentes() — GET Productos (Gzip) — base para IDs y SKUs
  → cargarPlantilla()        — GET S3 logigho-plantillas/Productos.xlsx

── PASO 1: Validar Excel ──
onArchivoSeleccionado()      — guarda File, limpia filasPreview
procesarExcel()
  → validar extensión (.xlsx / .csv)
  → leerExcel()              — FileReader → XLSX.read()
  → XLSX.utils.sheet_to_json
  → validar columnas con COLUMNAS_ESPERADAS
  → buildContexto()          — arma ContextoValidacion con tiendas/categorías/proveedores
  → validator.evaluarFila()  — por cada fila (Template Method)
  → filasPreview = [...]
avanzarAPaso2()
  → filasValidas = filasValidas_ (solo las que pasaron)
  → paso = 2

── PASO 2: Asignación de imágenes ──
onImagenSeleccionada()       — guarda File + genera URL.createObjectURL para preview
omitirImagen()               — limpia imagenFile y avanza al siguiente
avanzarAPaso3()
  → paso = 3
  → ejecutarImportacion()

── PASO 3: Importación ──
ejecutarImportacion() [por cada fila válida]
  → cargarProductosExistentes()    — consulta fresca para evitar race condition
  → obtenerIdConsecutivo()         — max(id) + 1
  → obtenerIdProductoConsecutivo() — max(idproducto) + 1
  → SKU dedup (Set local + cross-check BD)
  → subir imagen a S3 (si aplica) → URL CloudFront
  → insertarGenerico (colección Productos)
  → insertarGenerico (actualizaInventarioLotes)
  → resumen.push(...)
→ refreshDataEvent.emit() si hay al menos 1 creado
```

---

## Historial de cambios

| Fecha | Autor | Cambio |
| ---------- | ------ | ------ |
| 2026-05-20 | Adalberto González | Creación del componente con stepper de 3 pasos (validar Excel → imágenes → resumen). |
| 2026-05-20 | Adalberto González | Integración del patrón Template Method (`ProductoImportacionValidator`) para validación de filas. |
| 2026-05-20 | Adalberto González | Validación de columnas del Excel con `COLUMNAS_ESPERADAS` antes de procesar filas. |
| 2026-05-20 | Adalberto González | Columna `NombreCategoria` en Excel: el usuario elige por nombre, el sistema resuelve al ID internamente. |
| 2026-05-20 | Adalberto González | Regla 1 activa: `precioventa > precioproveedor`. Regla 2 activa: `IdTienda` debe pertenecer a las tiendas asignadas al usuario. |
| 2026-05-20 | Adalberto González | Race condition corregida: `id` e `idproducto` se recalculan con datos frescos de la BD antes de insertar cada producto. |
| 2026-05-20 | Adalberto González | SKU deduplicado con `Set` local + cross-check contra BD para evitar colisiones en el mismo lote. |
| 2026-05-20 | Adalberto González | Cierre del modal al hacer clic fuera del contenido (`modal-overlay` + `stopPropagation`). |
| 2026-05-20 | Adalberto González | Botón "Limpiar archivo": resetea `archivoSeleccionado`, `filasPreview` y el valor del input nativo para permitir re-subir un Excel corregido sin cerrar el modal. |

---

## Observaciones

- **Race condition mitigada, no eliminada:** La consulta fresca antes de cada inserción reduce la ventana de colisión a milisegundos, pero la solución definitiva es que el backend genere y garantice los IDs atómicamente. Ver [ADR](../views/tienda/portafolio/portafolio-ADR.md) para contexto.
- **Plantilla desde S3:** Si `cargarPlantilla()` falla (sin conexión o bucket inaccesible), el botón "Descargar plantilla" queda silencioso (no muestra Swal). Hay un `Swal.fire` comentado que puede reactivarse si se prefiere feedback explícito.
- **Compresión diferenciada:** Tiendas y proveedores usan Zstd (`mcomp: '2'`), categorías y productos usan Gzip estándar. Seguir este patrón al agregar nuevas fuentes de datos.
- **`cantidad` se guarda como `String`:** Consistencia con el tipo del campo en MongoDB. El resto de campos numéricos se guardan como `Number`.
- **Template Method ubicado en `portafolio/importacion-template-method/`:** Para agregar una nueva regla de negocio, editar solo `validarNegocio()` en `ProductoImportacionValidator`. Ver `ADR-001-template-method-importacion-masiva.md`.
