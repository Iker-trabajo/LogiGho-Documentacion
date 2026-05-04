---
autor: Adalberto González
fecha_creacion: 2026-04-28
ultima_actualizacion: 2026-05-02
estado: desarrollo
nivel: 1
---

# Importación Pagos Genérico

**Autor:** Adalberto González

**Selector:** `app-importacion-pagos-generico`

**Ubicación:** `SitioLogiGho/src/app/components/importacion-pagos-generico`

---

## ¿Qué hace?

Modal reutilizable para importar pagos de cualquier transportadora al sistema. Recibe una `TransportadoraPagoStrategy` como `@Input()` y usa su configuración para descargar la plantilla Excel desde S3, leer el archivo subido por el usuario, validar sus columnas, mapear sus campos, transformar valores especiales (fechas seriales de Excel) e insertar los registros en la colección MongoDB correspondiente por lotes de 5000.

A diferencia de `importacion-facturas-generico`, este componente agrega automáticamente un campo `IdCargaPago` secuencial por lote de importación, consultando el máximo existente en la colección antes de insertar.

---

## Roles y acceso

| Acceso | Descripción |
|---|---|
| Autenticado | Requiere login |
| Rol específico | Requiere rol: `controller` |

---

## Estructura de archivos

```
importacion-pagos-generico/
├── importacion-pagos-generico.component.ts
├── importacion-pagos-generico.component.html
└── importacion-pagos-generico.component.scss
```

---

## Inputs y Outputs

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `strategy` | `TransportadoraPagoStrategy` | requerido | Configuración y lógica de la transportadora: colección, campos, plantilla, mapeo, transformaciones |
| `isOpen` | `boolean` | `false` | Controla si el modal está visible. Al pasar a `true` descarga la plantilla automáticamente |
| `closeEvent` | `EventEmitter<void>` | — | Emitido cuando el modal se cierra (por el usuario o al finalizar la inserción) |
| `refreshDataEvent` | `EventEmitter<void>` | — | Emitido al finalizar una inserción exitosa para que el padre recargue sus datos |

---

## Propiedades internas

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `form` | `FormGroup` | `{ archivo: '' }` | Formulario reactivo que controla el input de archivo |
| `isLoading` | `boolean` | `false` | Muestra el estado de carga durante la descarga de plantilla o inserción |
| `fileToUpload` | `File \| null` | `null` | Archivo Excel seleccionado por el usuario |
| `uploadProgress` | `number` | `0` | Porcentaje de progreso de inserción (0–100) |
| `currentBatch` | `number` | `0` | Número de lote que se está procesando actualmente |
| `totalBatches` | `number` | `0` | Total de lotes calculados según el tamaño del archivo |
| `fileName` | `string` | `''` | Nombre del archivo de plantilla descargado desde S3 |
| `fileBlob` | `Blob` | — | Contenido binario de la plantilla para permitir su descarga |

---

## Métodos

### `ngOnChanges(changes: SimpleChanges): void`

**Descripción:** Detecta cuando `isOpen` cambia a `true` y dispara la descarga de la plantilla automáticamente.

**Proceso:**
1. Detecta cambio en `isOpen`
2. Si el nuevo valor es `true`, llama a `obtenerPlantilla()`

---

### `obtenerPlantilla(): void`

**Descripción:** Descarga la plantilla Excel desde el bucket S3 `logigho-plantillas` usando el nombre definido en `strategy.plantillaFileName`.

**Proceso:**
1. Activa `isLoading = true`
2. Llama a `GetObjectService.obtenerObjeto()` con bucket `logigho-plantillas` y `strategy.plantillaFileName`
3. Descomprime la respuesta con `DecompressionService` y convierte el base64 a `Blob`
4. Guarda el blob en `fileBlob` para descarga posterior
5. Desactiva `isLoading`

---

### `onPlantillaClick(): void`

**Descripción:** Permite al usuario descargar la plantilla Excel en su equipo creando un enlace temporal con el `Blob` guardado.

---

### `onFileChangeCarga(event: any): void`

**Descripción:** Captura el archivo seleccionado en el input y lo asigna a `fileToUpload`.

**Parámetros:**
- `event` — Evento del input `type="file"`

---

### `uploadFile(): Promise<void>`

**Descripción:** Procesa el archivo Excel subido por el usuario, lo valida y lanza la inserción en MongoDB.

**Proceso:**
1. Valida que existe un archivo seleccionado
2. Lee el archivo con `FileReader` en modo binario
3. Parsea el Excel con XLSX y extrae encabezados y filas
4. Valida que las columnas del Excel coincidan exactamente con las keys de `strategy.columnMapping`:
   - Si hay columnas faltantes o no reconocidas → muestra `'Documento no válido'` y aborta
5. Mapea los encabezados del Excel a campos de MongoDB usando `strategy.columnMapping`
6. Filtra filas vacías y filas cuya primera celda sea exactamente `"total"` (case-insensitive)
7. Fuerza a string el campo definido en `strategy.campoGuia` para preservar ceros iniciales
8. Aplica `strategy.transformarValor()` si está definido (ej: conversión de fecha serial de Excel a string)
9. Consulta el `IdCargaPago` máximo existente en la colección y asigna el siguiente a todos los registros del lote
10. Llama a `procesarInsercionPorLotes()` con los registros mapeados

---

### `close(): void`

**Descripción:** Cierra el modal, resetea el formulario y emite `closeEvent` al componente padre.

---

### `resetFileInput(): void`

**Descripción:** Limpia el valor del input de archivo en el DOM para permitir subir el mismo archivo nuevamente.

---

### `base64ToBlob(base64: string, mimeType: string): Blob` *(privado)*

**Descripción:** Convierte un string base64 (recibido del API) a un objeto `Blob` descargable.

**Parámetros:**
- `base64` — String base64 con prefijo separado por `|`
- `mimeType` — Tipo MIME del archivo resultante

**Retorna:** `Blob` con el contenido binario del archivo

---

### `obtenerSiguienteIdCargaPago(): Promise<number>` *(privado)*

**Descripción:** Consulta la colección de la transportadora actual y retorna el siguiente `IdCargaPago` secuencial.

**Proceso:**
1. Consulta todos los registros de la colección con `consultarGenerico()`
2. Descomprime la respuesta con `DecompressionService`
3. Busca el valor máximo del campo `IdCargaPago` en los documentos existentes
4. Retorna `máximo + 1`. Si la colección está vacía o falla la consulta, retorna `1`

---

### `procesarInsercionPorLotes(registros: any[], coleccion: string): Promise<void>` *(privado)*

**Descripción:** Inserta los registros en MongoDB en lotes de 5000 de forma recursiva y secuencial, actualizando el progreso en cada lote.

**Parámetros:**
- `registros` — Array completo de objetos a insertar
- `coleccion` — Endpoint con la colección destino (`metodoGenerico?coleccion=ConciliacionPagosD2E`)

**Proceso:**
1. Calcula el total de lotes (`Math.ceil(registros.length / 5000)`)
2. Por cada lote: llama a `insertarGenerico()`, actualiza `uploadProgress` y llama al siguiente lote de forma recursiva
3. Al terminar todos los lotes: muestra alerta de éxito, emite `refreshDataEvent`, resetea el input y cierra el modal
4. En caso de error: muestra alerta, resetea el progreso y el input

---

## Flujo principal

```
Usuario abre modal (isOpen = true)
  → ngOnChanges()
    → obtenerPlantilla()            → descarga plantilla desde S3

Usuario selecciona archivo
  → onFileChangeCarga()             → guarda archivo en fileToUpload

Usuario hace click en "Importar"
  → uploadFile()
    → FileReader.readAsBinaryString()
      → XLSX.read()                 → parsea el Excel
      → validar columnas            → aborta si no coinciden con columnMapping
      → mapear columnas             → strategy.columnMapping
      → filtrar filas vacías y "total"
      → transformarValor()          → normaliza fechas seriales si strategy lo implementa
      → obtenerSiguienteIdCargaPago() → IdCargaPago = max + 1
      → procesarInsercionPorLotes()
          → lote 1 → insertarGenerico() → uploadProgress 33%
          → lote 2 → insertarGenerico() → uploadProgress 66%
          → lote N → insertarGenerico() → uploadProgress 100%
          → Swal éxito → refreshDataEvent → close()
```

---

## Dependencias externas

| Librería | Versión | Uso |
|---|---|---|
| XLSX (SheetJS) | ^0.18 | Lectura y parseo de archivos Excel |
| sweetalert2 | ^11.x | Alertas de éxito, advertencia y error |

## Servicios Angular utilizados

| Servicio | Métodos usados | Propósito |
|---|---|---|
| `ConsumoGenericoService` | `insertarGenerico()`, `consultarGenerico()` | Inserción y consulta de registros en MongoDB |
| `GetObjectService` | `obtenerObjeto()` | Descarga de la plantilla Excel desde S3 |
| `DecompressionService` | `decompressGzip()` | Descompresión de respuestas gzip del backend |
| `FormBuilder` | `group()` | Construcción del formulario reactivo |

---

## Estilos

> Este componente hereda los estilos globales del sistema. No define variables SCSS propias.

### Animaciones

| Animación | Duración | Uso |
|---|---|---|
| `slide-in` | 0.5s ease | Entrada del modal desde arriba |

---

## Subcomponentes

> Este componente no tiene subcomponentes. Es un componente hoja.

---

## Cómo usarlo desde un componente padre

```typescript
// En el .ts del padre
readonly d2eStrategy = TransportadoraPagoFactory.crear('d2e');
isModalOpen = false;
openModal()  { this.isModalOpen = true;  }
closeModal() { this.isModalOpen = false; }
recargarDatos() { /* lógica de recarga */ }
```

```html
<!-- En el .html del padre -->
<app-importacion-pagos-generico
  [strategy]="d2eStrategy"
  [isOpen]="isModalOpen"
  (closeEvent)="closeModal()"
  (refreshDataEvent)="recargarDatos()">
</app-importacion-pagos-generico>

<button (click)="openModal()">Cargar Pagos</button>
```

---

## Diferencias respecto a `importacion-facturas-generico`

| Aspecto | `importacion-pagos-generico` | `importacion-facturas-generico` |
|---|---|---|
| Strategy | `TransportadoraPagoStrategy` | `TransportadoraStrategy` |
| `IdCargaPago` | Sí — secuencial por colección | No |
| Validación de columnas | Sí — aborta si no coinciden | Sí — aborta si no coinciden |
| `transformarValor()` | Sí — normaliza fechas seriales | Sí — normaliza fechas seriales |
| Filtro fila "total" | Sí | Sí |
| `refreshDataEvent` | Emitido al finalizar inserción | Declarado, no emitido |
| `DecompressionService` | Sí (para `IdCargaPago` y plantilla) | No |

---

## Changelog del componente

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-04-28 | Adalberto González | Creación del componente genérico como parte de la implementación del patrón Strategy para transportadoras de pagos |
| 2026-05-02 | Adalberto González | Validación estricta de columnas contra `columnMapping`, llamada a `transformarValor()`, filtro de filas "total" |
