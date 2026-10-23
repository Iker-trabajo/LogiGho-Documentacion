---
autor: Iker
fecha_creacion: 2026-03-23
ultima_actualizacion: 2026-10-02
estado: desarrollo
nivel: 1
---

# Importación Facturas Genérico

**Autor:** Iker

**Selector:** `app-importacion-facturas-generico`

**Ubicación:** `SitioLogiGho/src/app/components/importacion-facturas-generico`

---

## ¿Qué hace?

Modal reutilizable para importar facturas de cualquier transportadora al sistema. Recibe una `TransportadoraStrategy` como `@Input()` y usa su configuración para descargar la plantilla Excel desde S3, leer el archivo subido por el usuario, mapear sus columnas y insertar los registros en la colección MongoDB correspondiente por lotes de 5000.

Este componente reemplaza el patrón de un componente de importación por transportadora. Cualquier transportadora nueva que use el patrón Strategy lo reutiliza sin modificarlo.

---

## Roles y acceso

| Acceso | Descripción |
|---|---|
| Autenticado | Requiere login |
| Rol específico | Requiere rol: `controller` |

---

## Estructura de archivos

```
importacion-facturas-generico/
├── importacion-facturas-generico.component.ts
├── importacion-facturas-generico.component.html
└── importacion-facturas-generico.component.scss
```

---

## Inputs y Outputs

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `strategy` | `TransportadoraStrategy` | requerido | Configuración y lógica de la transportadora: colección, campos, plantilla, mapeo |
| `isOpen` | `boolean` | `false` | Controla si el modal está visible. Al pasar a `true` descarga la plantilla automáticamente |
| `closeEvent` | `EventEmitter<void>` | — | Emitido cuando el modal se cierra (por el usuario o al finalizar la inserción) |
| `refreshDataEvent` | `EventEmitter<void>` | — | Disponible para que el padre recargue sus datos tras una inserción exitosa |

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
3. Convierte la respuesta base64 a `Blob` con `base64ToBlob()`
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

### `uploadFile(): void`

**Descripción:** Procesa el archivo Excel subido por el usuario y lanza la inserción en MongoDB.

**Proceso:**
1. Valida que existe un archivo seleccionado
2. Lee el archivo con `FileReader` en modo binario
3. Parsea el Excel con XLSX y extrae encabezados y filas
4. Ejecuta la [Validación 1](#validacion-de-encabezados) (primera fila de encabezados)
5. Ejecuta la [Validación 2](#validacion-de-encabezados) (nombres de columnas y obligatorios, según `strategy.columnMapping`)
6. Mapea los encabezados del Excel a campos de MongoDB usando `strategy.columnMapping`
7. Filtra filas vacías
8. Fuerza a string el campo definido en `strategy.campoGuia` para preservar ceros iniciales (ej: `IDFACTURA: "0005090781"`)
9. Llama a `procesarInsercionPorLotes()` con los registros mapeados

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

### `procesarInsercionPorLotes(registros: any[], coleccion: string): Promise<void>` *(privado)*

**Descripción:** Inserta los registros en MongoDB en lotes de 5000 de forma recursiva y secuencial, actualizando el progreso en cada lote.

**Parámetros:**
- `registros` — Array completo de objetos a insertar
- `coleccion` — Endpoint con la colección destino (`metodoGenerico?coleccion=FacturacionServientrega`)

**Proceso:**
1. Calcula el total de lotes (`Math.ceil(registros.length / 5000)`)
2. Por cada lote: llama a `insertarGenerico()`, actualiza `uploadProgress` y llama al siguiente lote de forma recursiva
3. Al terminar todos los lotes: muestra alerta de éxito, resetea el input y cierra el modal
4. En caso de error: muestra alerta, resetea el progreso y el input

---

## Validación de encabezados

Se ejecuta en `uploadFile()`, **antes** de mapear las columnas. Si cualquiera de las dos validaciones falla, el componente: quita el spinner (`isLoading = false`), limpia el archivo seleccionado (`resetFileInput()`), muestra una alerta con `Swal` y **detiene el proceso: no se inserta nada en MongoDB**. Además deja un `console.warn` con lo que falló (visible al inspeccionar la consola).

| # | Validación | Qué comprueba | Alerta si falla |
|---|---|---|---|
| 1 | Primera fila de encabezados | Todas las celdas de la primera fila deben ser texto y no estar vacías | `Archivo sin encabezados` |
| 2 | Nombres y obligatorios | Cada encabezado debe existir en `strategy.columnMapping` y deben venir todas las columnas que la strategy no marque como opcionales | `Encabezados incorrectos`, con las listas **No reconocidos** (sin repetidos) y **Faltan obligatorios** |

### Columnas opcionales

La lista de columnas **no está escrita en el componente**: sale de `this.strategy.columnMapping`. Cada transportadora puede declarar en su strategy qué columnas no son obligatorias con el campo opcional `columnasOpcionales`:

| Transportadora | `columnasOpcionales` | Obligatorias |
|---|---|---|
| Servientrega | `CODIGOCENTROCOSTO`, `NOMBRECENTROCOSTO` (confirmado con el responsable) | Las otras 29 columnas |
| D2E | No la define | Sus 9 columnas |

Si la strategy no define `columnasOpcionales`, todas las columnas de su `columnMapping` son obligatorias. Las opcionales no cuentan como faltantes si no vienen; si vienen, se guardan normal.

### Código

```typescript
/** Validación 2: los encabezados deben coincidir con strategy.columnMapping.
 * Cada transportadora define sus columnas obligatorias en su strategy.
 * Si falta alguna o aparece una columna desconocida, se rechaza el archivo. */
const columnasEsperadas = Object.keys(this.strategy.columnMapping);
const columnasOpcionales = this.strategy.columnasOpcionales ?? [];
// Obligatorias que no vinieron en el Excel (las opcionales no cuentan como faltantes)
const columnasFaltantes = columnasEsperadas.filter(col => !columnasOpcionales.includes(col) && !headers.includes(col));
// Encabezados del Excel que no están en el columnMapping (sin repetidos)
const columnasExtra = Array.from(new Set(headers.filter(h => !this.strategy.columnMapping[h])));

if (columnasFaltantes.length > 0 || columnasExtra.length > 0) {
  console.warn('Validación de encabezados falló. No reconocidos:', columnasExtra, '| Faltantes:', columnasFaltantes);
  this.isLoading = false;
  this.resetFileInput();
  let detalle = 'Los nombres de las columnas no coinciden con la plantilla.';
  if (columnasExtra.length > 0) {
    detalle += `<br><br><b>No reconocidos:</b> ${columnasExtra.map(h => `"${h}"`).join(', ')}`;
  }
  if (columnasFaltantes.length > 0) {
    detalle += `<br><br><b>Faltan obligatorios:</b> ${columnasFaltantes.join(', ')}`;
  }
  Swal.fire({ title: 'Encabezados incorrectos', html: detalle, icon: 'error' });
  return;
}
```

La Validación 1 es idéntica a la de [Importación Facturas Envía](importacion-facturas-envia.md#codigo).

El campo se declara en la interfaz `transportadora.strategy.ts`:

```typescript
/**
 * Opcional: columnas del columnMapping que NO son obligatorias en el Excel.
 * Si el archivo no las trae, la validacion de encabezados no lo rechaza; si las trae, se guardan normal.
 * Si la transportadora no define esta lista, todas las columnas del columnMapping son obligatorias.
 */
columnasOpcionales?: string[];
```

Y se usa en `servientrega.strategy.ts`, justo antes de `sanitize`:

```typescript
/**
 * Columnas que NO son obligatorias en el Excel de Servientrega (confirmado con el responsable).
 * El resto de columnas del columnMapping si son obligatorias en la validacion de encabezados.
 */
columnasOpcionales: string[] = ['CODIGOCENTROCOSTO', 'NOMBRECENTROCOSTO'];
```

---

## Flujo principal

```
Usuario abre modal (isOpen = true)
  → ngOnChanges()
    → obtenerPlantilla()   → descarga plantilla desde S3

Usuario selecciona archivo
  → onFileChangeCarga()    → guarda archivo en fileToUpload

Usuario hace click en "Cargar"
  → uploadFile()
    → FileReader.readAsBinaryString()
      → XLSX.read()        → parsea el Excel
      → Validación 1       → ¿primera fila de encabezados?  (si falla: alerta y se detiene)
      → Validación 2       → ¿nombres válidos y obligatorios? (si falla: alerta y se detiene)
      → mapear columnas    → strategy.columnMapping
      → filtrar vacíos
      → procesarInsercionPorLotes()
          → lote 1 → insertarGenerico() → uploadProgress 33%
          → lote 2 → insertarGenerico() → uploadProgress 66%
          → lote N → insertarGenerico() → uploadProgress 100%
          → Swal éxito → close()
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
| `ConsumoGenericoService` | `insertarGenerico()` | Inserción de registros en MongoDB |
| `GetObjectService` | `obtenerObjeto()` | Descarga de la plantilla Excel desde S3 |
| `FormBuilder` | `group()` | Construcción del formulario reactivo |

---

## Estilos

> Este componente hereda los estilos globales del sistema. No define variables SCSS propias.

### Animaciones

| Animación | Duración | Uso |
|---|---|---|
| — | — | Sin animaciones propias |

---

## Subcomponentes

> Este componente no tiene subcomponentes. Es un componente hoja.

---

## Cómo usarlo desde un componente padre

```typescript
// En el .ts del padre
readonly miStrategy = TransportadoraFactory.crear('servientrega');
isModalOpen = false;
openModal()  { this.isModalOpen = true;  }
closeModal() { this.isModalOpen = false; }
```

```html
<!-- En el .html del padre -->
<app-importacion-facturas-generico
  [strategy]="miStrategy"
  [isOpen]="isModalOpen"
  (closeEvent)="closeModal()">
</app-importacion-facturas-generico>

<button (click)="openModal()">Cargar Facturas</button>
```

---

## Changelog del componente

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-03-23 | Iker | Creación del componente genérico como parte de la implementación del patrón Strategy para transportadoras |
| 2026-05-02 | Adalberto | Validación estricta de columnas en `importacion-facturas-generico`: si el archivo Excel tiene columnas faltantes o no reconocidas respecto a `columnMapping`, se muestra `'Documento no válido'` y se aborta la inserción sin tocar MongoDB. |
| 2026-05-02 | Adalberto | Filtro de fila totales en `importacion-facturas-generico`: se descartan filas donde cualquier celda sea exactamente `'total'` (case-insensitive) antes de insertar. |
| 2026-10-02 | Yadilton Torres | Agregada la Validación 1 (la primera fila debe ser de encabezados: texto y sin celdas vacías → alerta `Archivo sin encabezados`). La validación anterior (`'Documento no válido'`) se reemplazó por la Validación 2 con mensaje detallado (`Encabezados incorrectos`: no reconocidos sin repetidos y faltantes). |
| 2026-10-02 | Yadilton Torres | Soporte de columnas opcionales: las columnas listadas en `strategy.columnasOpcionales` no cuentan como faltantes. Si la strategy no define la lista (caso D2E), todas sus columnas son obligatorias. |

---

## Observaciones

- **Cambio de comportamiento:** un archivo con columnas fuera de `strategy.columnMapping` se rechaza (ya ocurría desde 2026-05-02); ahora el mensaje indica cuáles columnas no se reconocen y cuáles faltan. Probar con archivos reales de Servientrega y D2E.
- El componente asume que la plantilla existe en el bucket S3 `logigho-plantillas` con el nombre exacto definido en `strategy.plantillaFileName`. Si no existe, mostrará una advertencia pero no bloqueará el flujo de carga.
- El tamaño de lote está fijado en 5000 registros (`TAMANO_LOTE = 5000`). Si se necesita ajustar, modificar la constante en `procesarInsercionPorLotes`.
- `refreshDataEvent` está declarado pero no está siendo emitido actualmente. El componente padre recarga los datos vía el evento `(updated)` del componente `app-tables`. Pendiente evaluar si se necesita.
