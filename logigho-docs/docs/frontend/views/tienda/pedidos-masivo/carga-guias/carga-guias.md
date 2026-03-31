---
autor: Iker
fecha_creacion: 2026-03-31
ultima_actualizacion: 2026-03-31
estado: desarrollo
nivel: 2
---

# Vista: Carga Masiva de Guias

**Autor:** Iker

**Selector:** `app-carga-guias`

**Ubicación:** `SitioLogiGho/src/app/views/tienda/pedidos-masivo/carga-guias`

---

## ¿Qué hace?

Permite a los usuarios de cada tienda cargar de forma masiva sus guías de pedido a través de un archivo Excel (`.xlsx`). El componente descarga la plantilla oficial desde S3, valida y procesa el archivo subido, agrupa los pedidos por tienda, asigna IDs únicos y los registra en la base de datos. Para tiendas tipo Dropshipping también valida stock antes de insertar.

---

## Ruta

| Propiedad | Valor |
|---|---|
| **Ruta** | `/tienda/carga-guias-masivo` |
| **Título de página** | `Carga-guias` |
| **Rol requerido** | Usuarios con tiendas asignadas (`tiendas_asignadas` en `sessionStorage`) |


### Definición en `routes.ts`

```typescript
{
  path: 'carga-guias-masivo',
  loadComponent: () =>
    import('./pedidos-masivo/carga-guias/carga-guias.component')
      .then(m => m.CargaGuiasComponent),
  data: { title: 'Carga-guias' }
}
```

---

## Estructura de archivos

```
carga-guias/
├── carga-guias.component.ts       Lógica principal
├── carga-guias.component.html     Template de la vista
├── carga-guias.component.scss     Estilos propios con variables SCSS
└── carga-guias.component.spec.ts  Pruebas unitarias

workers/  (compartido con otros componentes del módulo pedidos-masivo)
└── carga-guias.worker.ts          Web Worker: parseo del Excel con SheetJS
```

---

## Secciones de la vista

| # | Sección | Descripción |
|---|---|---|
| 1 | **Encabezado** | Header con gradiente azul, ícono de documento, título y decoración SVG |
| 2 | **Recomendaciones** | Panel acordeón colapsable con 4 tips antes de cargar (plantilla, formato celdas, campos obligatorios, límite de registros) |
| 3 | **Paso 1 – Descargar plantilla** | Muestra el archivo `carga-masiva.xlsx` con botón de descarga |
| 4 | **Paso 2 – Subir archivo** | Área de clic con visualización del archivo seleccionado, botón "Cargar Guias" |
| 5 | **Resumen de guias cargadas** | Tabla paginada con los pedidos procesados y badges de estado. Solo visible si `rows.length > 0` |
| 6 | **Estado vacío** | Mensaje orientativo cuando no hay datos cargados. Solo visible si `rows.length === 0` y no está cargando |

---

## Propiedades del componente

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `extensionArchivo` | `string` | `""` | Extensión del archivo plantilla descargado desde S3 |
| `fileName` | `string` | `''` | Nombre completo del archivo plantilla (ej. `carga-masiva.xlsx`) |
| `fileBlob` | `Blob` | — | Contenido binario de la plantilla cargada en memoria al iniciar |
| `isLoading` | `boolean` | `false` | Activa el overlay de carga full-screen con spinner |
| `fileToUpload` | `File \| null` | `null` | Archivo Excel seleccionado por el usuario para procesar |
| `rows` | `TablaRow[]` | `[]` | Filas de pedidos para mostrar en la tabla de resultados |
| `columns` | `TablaColumn[]` | `[]` | Columnas generadas dinámicamente desde los datos del Excel |
| `isButtonDisabled` | `boolean` | `false` | Deshabilita el botón "Cargar Guias" durante el procesamiento |
| `tipsExpanded` | `boolean` | `true` | Controla si el panel de recomendaciones está abierto o cerrado |
| `currentPage` | `number` | `1` | Página activa en la tabla de resultados |
| `pageSize` | `number` | `50` | Registros por página en la tabla |

### Constantes privadas

| Constante | Valor | Descripción |
|---|---|---|
| `MAX_FILE_SIZE_BYTES` | `10 * 1024 * 1024` (10 MB) | Tamaño máximo permitido para el archivo subido |
| `ALLOWED_EXTENSION` | `'xlsx'` | Única extensión de archivo aceptada |
| `FIELDS_TO_EXCLUDE` | Ver lista abajo | Columnas del Excel ignoradas en la deduplicación y no enviadas al backend como datos del pedido |

**Campos excluidos (`FIELDS_TO_EXCLUDE`):**
`Fecha Carga`, `Estado Logigho`, `Sub Estado Logigho`, `Estado Financiero Logigho`, `Compra Logigho`, `Retorno`, `Fecha Despacho`, `Fecha Entrega`, `Fecha Devolucion`, `Fecha Pago`, `Pago`, `Porcentaje Fee`, `Fee`, `Compra`, `Estado financiero Tienda`, `Sub Estado Financiero Logigho`

### Getters

| Getter | Tipo retorno | Descripción |
|---|---|---|
| `paginatedRows` | `TablaRow[]` | Calcula y retorna las filas de la página actual según `currentPage` y `pageSize` |
| `totalPages` | `number` | Total de páginas: `Math.ceil(rows.length / pageSize)` |

### Output / EventEmitter

| Output | Tipo | Descripción |
|---|---|---|
| `refreshDataEvent` | `EventEmitter<void>` | Emite una señal al componente padre para que refresque sus datos. Actualmente declarado pero no emitido en el flujo visible |

---

## Flujo de inicialización

```
ngOnInit()
  └─> obtenerArchivo()
        └─> GetObjectService.obtenerObjeto('logigho-plantillas', 'carga-masiva.xlsx', ...)
              └─> S3 GET logigho-plantillas/carga-masiva.xlsx
              → base64ToBlob() → guarda en this.fileBlob (en memoria)
              → guarda this.fileName = 'carga-masiva.xlsx'
```

---

## Flujo principal: `uploadFile()`

Este es el método central del componente. Se ejecuta al hacer clic en "Cargar Guias".

```
uploadFile()
  1. Verifica sessionStorage['tiendas_asignadas'] → si vacío, muestra Swal y aborta
  2. Verifica this.fileToUpload !== null → si no hay archivo, muestra Swal y aborta
  3. validarArchivoBasico() → valida extensión (.xlsx) y tamaño (≤ 10 MB)
  4. leerArchivoComoBuffer() → FileReader → ArrayBuffer
  5. procesarExcelEnWorker(buffer)
        └─> Web Worker (carga-guias.worker.ts con SheetJS)
              → parsea Excel, elimina filas vacías y duplicados
              → agrupa pedidos por columna 'ID TIENDA'
              → devuelve Map<tiendaId, pedidos[]> y totalPedidos
  6. obtenerFechaFormateada() → fecha actual UTC-5 como "YYYY-MM-DD HH:MM:SS"
  7. generarIdCargaCompleta() → ID único: AAMMDDHHMMSSRAND (16 chars)
  8. Verifica pedidosPorTienda.size > 0
  9. obtenerNombresTiendas(tiendaIds)
        └─> TiendasService.consultarTienda()
              → descomprime con decompressionService.decompressGzip()
              → retorna Map<tiendaId, { nombre, tipoTienda[] }>
 10. filtrarTiendasInvalidas() → elimina del mapa las tiendas no encontradas en BD
                                → muestra Swal de advertencia por cada ID inválido
 11. prepararCargas() → construye array de objetos carga con metadata de tienda
 12. obtenerDatosCargaPedido()
        └─> ConsumoGenericoService.consultarGenerico('metodoGenerico?coleccion=CargaPedido')
              → descomprime con decompressionService.decompressZstd()
              → retorna { ultimoIdCarga, ultimoIdOrdenPedido, idsExistentes: Set<number> }
 13. asignarIdsCarga() → asigna IdCarga único a cada carga (salta IDs ya existentes en BD)
 14. enriquecerYPrepararPedidos()
        ├─> Si tipoTienda === ['Dropshipping']:
        │       └─> consultarGenericoAnidado('metodoGenerico?coleccion=Productos&...')
        │             → valida ID STOCK 1..12 y cantidades contra stock disponible
        │             → si hay errores: muestra Swal detallado y lanza Error('SILENT:...')
        └─> Para todos: construye pedidosConIds con campos:
              IdCargaCompleta, Registro (tiendaId-idCarga), IdCarga, IdOrdenPedido,
              Fecha Carga, Estado="Pendiente", IdTienda, Tienda, + todos los campos del Excel
              (sin ID TIENDA ni los campos excluidos de deduplicación)
              + TiendaProveedor (solo Dropshipping)
 15. ejecutarCargas(cargas)
        └─> ejecutarConConcurrencia(tareas, limite=3)
              └─> ConsumoGenericoService.insertarGenerico(pedidosConIds, 'cargaPedidos')
                    → POST /cargaPedidos con todos los pedidos de la tienda
                    → retorna { tienda, tiendaId, idCarga, estado: 'OK'|'ERROR', mensaje, ... }
 16. Renderiza tabla: this.columns, this.rows = todosLosPedidos
 17. mostrarResumenCargaMasiva() → Swal con tabla HTML de resultados por tienda
```

**Control de errores:** Los errores con prefijo `SILENT:` ya mostraron su propio Swal (validación de stock). El resto se captura en el `catch` del `try` principal y muestra un Swal genérico.

---

## Métodos

### Públicos

| Método | Descripción |
|---|---|
| `ngOnInit()` | Ciclo de vida Angular. Llama a `obtenerArchivo()` al iniciar |
| `obtenerArchivo()` | Descarga la plantilla desde S3 y la guarda como Blob en memoria |
| `descargarArchivo()` | Crea un enlace temporal y descarga `this.fileBlob` al equipo del usuario |
| `onFileChangeCarga(event)` | Captura el archivo seleccionado por el input y lo asigna a `fileToUpload` |
| `uploadFile()` | Orquesta el proceso completo de carga masiva (ver flujo arriba) |
| `generateColumns(data)` | Genera columnas dinámicas para la tabla a partir de los datos; procesa objetos anidados y arrays recursivamente |
| `capitalizeToPascalCase(str)` | Convierte `fecha_carga` → `FechaCarga`. Separa por `_` o espacios |
| `resetFileInput()` | Limpia el value del `<input type="file">` vía `ViewChild` |
| `generarIdCargaCompleta()` | Genera un ID único de lote en formato `AAMMDDHHMMSSRAND` (16 chars) |
| `agruparPorTienda(pedidos)` | Agrupa un arreglo de pedidos por `pedido['ID TIENDA']`; retorna `Map<string, any[]>` |
| `obtenerDatosCargaPedido()` | Consulta la colección `CargaPedido` y retorna los últimos IDs y el set de IDs existentes |
| `mostrarResumenCargaMasiva(resultados)` | Muestra un Swal con tabla HTML con el resultado por tienda (exitosas/fallidas) |
| `nextPage()` | Avanza a la siguiente página de la tabla si no es la última |
| `prevPage()` | Retrocede a la página anterior de la tabla si no es la primera |

### Privados

| Método | Descripción |
|---|---|
| `validarArchivoBasico(file)` | Valida extensión (solo `.xlsx`) y tamaño máximo (10 MB). Lanza `Error` si no cumple |
| `leerArchivoComoBuffer(file)` | Lee el archivo con `FileReader` y retorna `Promise<ArrayBuffer>` |
| `procesarExcelEnWorker(buffer)` | Delega el parseo del Excel a un Web Worker para no bloquear la UI. Retorna `{ pedidosPorTienda, totalPedidos }` |
| `obtenerFechaFormateada()` | Retorna la fecha actual ajustada a UTC-5 (Colombia) en formato `YYYY-MM-DD HH:MM:SS` |
| `convertirExcelAJson(data)` | Convierte el array 2D del Excel a JSON, omitiendo filas vacías y filas duplicadas |
| `filtrarTiendasInvalidas(...)` | Muta `pedidosPorTienda` eliminando las tiendas no encontradas en BD y muestra una advertencia |
| `prepararCargas(...)` | Construye el arreglo de cargas con metadata de tienda e `idCargaCompleta`. `idCarga = 0` como placeholder |
| `asignarIdsCarga(...)` | Asigna un `IdCarga` único a cada carga saltando IDs ya existentes en la BD |
| `enriquecerYPrepararPedidos(...)` | Para Dropshipping: consulta productos y valida stock. Para todos: construye `pedidosConIds` con todos los campos requeridos por el backend |
| `ejecutarCargas(cargas)` | Envía cada carga al backend. Un fallo en una carga no detiene las demás. Concurrencia máxima: 3 |
| `ejecutarConConcurrencia(tareas, limite)` | Ejecuta promesas en paralelo respetando el límite de concurrencia |
| `obtenerNombresTiendas(tiendaIds)` | Consulta el catálogo de tiendas y retorna un mapa solo con los IDs del Excel |
| `base64ToBlob(base64, mimeType)` | Decodifica un string Base64 (con formato `prefix\|base64data`) y retorna un `Blob` |
| `guessMimeTypeByFilename(filename)` | Infiere el tipo MIME por la extensión del archivo. Soporta: jpg, png, gif, pdf, xlsx, csv |

---

## Servicios utilizados

| Servicio | Métodos usados | Propósito |
|---|---|---|
| `TiendasService` | `consultarTienda()` | Obtiene el catálogo completo de tiendas para validar los IDs del Excel |
| `GetObjectService` | `obtenerObjeto(bucket, key, ...)` | Descarga la plantilla `carga-masiva.xlsx` desde S3 |
| `DecompressionService` | `decompressGzip()`, `decompressZstd()` | Descomprime respuestas del backend (Gzip para tiendas/productos, Zstd para CargaPedido) |
| `ConsumoGenericoService` | `consultarGenerico()`, `consultarGenericoAnidado()`, `insertarGenerico()` | Consultas y escritura genérica a la BD (CargaPedido, Productos) |

---

## Endpoints que consume

| Método | Destino / Ruta | Cuándo |
|---|---|---|
| `GET` | S3 bucket `logigho-plantillas` / `carga-masiva.xlsx` | Al iniciar el componente (`ngOnInit`) |
| `GET` | `metodoGenerico?coleccion=CargaPedido` | Al iniciar la carga, para obtener los últimos IDs registrados |
| `GET` | `metodoGenerico?coleccion=Productos&perfilproducto=Publico&Tienda=...` | Solo para tiendas Dropshipping, para validar stock |
| `GET` | Endpoint de tiendas (`TiendasService.consultarTienda`) | Para validar que los IDs de tienda del Excel existen en el sistema |
| `POST` | `cargaPedidos` | Por cada tienda en el Excel; inserta todos los pedidos de esa tienda |

---

## Web Worker

**Archivo:** `src/app/views/tienda/pedidos-masivo/workers/carga-guias.worker.ts`

El componente delega el parseo del Excel a un Web Worker para no bloquear el hilo principal de la UI. El proceso dentro del worker:

1. Recibe el `ArrayBuffer` del archivo Excel y la lista `fieldsToExclude`
2. Parsea el Excel con **SheetJS** (`xlsx` library)
3. Convierte el array 2D a JSON, eliminando filas vacías y duplicados
4. Agrupa los pedidos por columna `ID TIENDA`
5. Retorna `{ ok: true, pedidosPorTienda: [...entries], totalPedidos }` al componente principal
6. En caso de error retorna `{ ok: false, error: mensaje }`

El `ArrayBuffer` se transfiere (no se copia) al worker para mayor eficiencia de memoria.

---

## Estados de la vista

| Estado | Condición | Qué muestra |
|---|---|---|
| **Inicial** | `rows.length === 0`, `isLoading = false` | Recomendaciones, pasos 1 y 2, tarjeta de estado vacío |
| **Cargando plantilla** | `ngOnInit` en curso | No bloquea UI, la plantilla se descarga en segundo plano |
| **Procesando** | `isLoading = true` | Overlay fijo full-screen con spinner y texto "Procesando archivo..." |
| **Con datos** | `rows.length > 0` | Tabla paginada con los pedidos procesados y sus estados |
| **Error** | `catch` del `uploadFile` | Swal de error con mensaje descriptivo |
| **Advertencia tiendas** | Tiendas del Excel no encontradas en BD | Swal de advertencia listando los IDs omitidos; el proceso continúa con las válidas |
| **Error de stock** | Dropshipping con stock insuficiente | Swal detallado con teléfono y error por cada registro inválido; se aborta la carga |

---

## Validaciones del archivo

| Validación | Dónde ocurre | Comportamiento si falla |
|---|---|---|
| Extensión `.xlsx` | `validarArchivoBasico()` en el componente | Lanza `Error`, muestra Swal |
| Tamaño ≤ 10 MB | `validarArchivoBasico()` en el componente | Lanza `Error`, muestra Swal con el tamaño actual |
| Sin filas vacías | Web Worker | Las filas vacías se omiten silenciosamente |
| Sin duplicados | Web Worker (clave: todas las columnas no excluidas concatenadas con `\|`) | Los duplicados se omiten silenciosamente |
| Tiendas asignadas | `uploadFile()`, antes de procesar | Swal de advertencia, aborta la carga |
| IDs de tienda válidos | `filtrarTiendasInvalidas()` | Swal de advertencia, omite las tiendas inválidas |
| Stock Dropshipping | `enriquecerYPrepararPedidos()` | Swal con detalle de errores, aborta toda la carga |

---

## Lógica de IDs

El componente necesita asignar IDs únicos y no colisionantes con los ya existentes en BD.

- **`IdCargaCompleta`:** Identificador de lote de toda la carga. Formato: `AAMMDDHHMMSSRAND` (16 chars). Mismo para todos los pedidos de una misma operación de carga.
- **`IdCarga`:** Identificador por tienda dentro de la carga. Se obtiene consultando `CargaPedido` y tomando el máximo `IdCarga` existente +1, saltando IDs ya tomados.
- **`Registro`:** Concatenación `{tiendaId}-{idCarga}`. Identifica la subida de una tienda específica dentro del lote.
- **`IdOrdenPedido`:** Número secuencial por pedido. Se obtiene consultando el máximo `IdOrdenPedido` en BD y se incrementa uno por uno por cada pedido procesado.

---

## Navegación desde esta vista

Esta vista no realiza redirecciones. Es un componente autocontenido del módulo de pedidos masivo.

---

## Estilos

El componente usa SCSS propio con variables definidas en el archivo `.scss`.

| Variable | Valor | Uso |
|---|---|---|
| `$primary-blue` | `#1e40af` | Color principal de botones, cabeceras, textos de sección |
| `$secondary-blue` | `#3b82f6` | Hover del área de carga, links |
| `$gradient-primary` | `linear-gradient(135deg, #1e40af → #1e3a8a)` | Header y botón principal |
| `$success` | `#10b981` | Badge OK, borde del área con archivo seleccionado |
| `$warning` | `#f59e0b` | Badge Pendiente |
| `$danger` | `#ef4444` | Badge ERROR/Devuelto, botón eliminar archivo |

**Responsive:**
- `≤ 768px`: Grid de 1 columna para los pasos 1 y 2, ícono del header oculto, botones a full width
- `≤ 480px`: Border-radius reducido, padding disminuido, fuentes más pequeñas

---

## Changelog de la vista

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-03-31 | Iker | Mejora en rendimiento: parseo del Excel movido a Web Worker para no bloquear UI |
| 2026-03-xx | Iker | Mejoras generales en módulo pedidos masivo, optimización de performance |
| Anterior | Iker | Modularización del código del módulo pedidos masivo |

---

## Observaciones

- **Web Worker obligatorio:** El parseo de archivos Excel grandes (hasta 5000 filas) se delega completamente al worker. Mover esa lógica de vuelta al componente principal bloquearía el hilo de UI.
- **Concurrencia 3 en cargas:** Si hay pedidos de más de 3 tiendas, las cargas se ejecutan en lotes de 3 simultáneas. Un fallo en una tienda no cancela las demás.
- **Deduplicación por llave compuesta:** Los duplicados se detectan concatenando los valores de todas las columnas que NO están en `FIELDS_TO_EXCLUDE`. Esto significa que dos pedidos idénticos en los campos de negocio se consideran duplicados aunque difieran en campos de estado/fecha.
- **Dropshipping exclusivo:** La validación de stock (`ID STOCK 1..12`, `CANTIDAD STOCK 1..12`) solo aplica cuando `tipoTienda` es exactamente `['Dropshipping']` (arreglo de un solo elemento). Si la tienda tiene varios tipos, no se valida stock.
- **`refreshDataEvent` no emitido:** El `@Output()` está declarado pero actualmente no se usa en el flujo visible del componente. Es probable que esté preparado para cuando este componente se use embebido en otro.
- **Plantilla en memoria:** La plantilla `carga-masiva.xlsx` se descarga al iniciar y se guarda en `this.fileBlob`. Si la descarga falla, el usuario ve un Swal de advertencia pero el resto de la vista sigue funcional (no puede descargar la plantilla, pero sí subir archivos que ya tenga).
- **Zona horaria hardcodeada:** La fecha de carga resta 5 horas fijas para UTC-5 (Colombia). No usa detección automática de zona horaria.
