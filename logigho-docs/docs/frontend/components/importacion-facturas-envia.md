---
autor: Yadilton Torres
fecha_creacion: 2026-10-02
ultima_actualizacion: 2026-10-02
estado: desarrollo
nivel: 1
---

# Importación Facturas Envía

**Autor:** Yadilton Torres

**Selector:** `app-importacion-facturas-envia`

**Ubicación:** `SitioLogiGho/src/app/components/importacion-facturas-envia`

---

## ¿Qué hace?

Modal para importar las facturas de Envía desde un archivo Excel. Descarga la plantilla desde S3, lee el archivo subido por el usuario, **valida los encabezados**, mapea las columnas y inserta los registros en la colección MongoDB `FacturacionEnvia` por lotes de 5000.

Es un componente **legacy**: tiene su propio `columnMapping` escrito dentro de `uploadFile()` y no usa el patrón Strategy (ver [Facturas Transportadoras](../views/controller/facturas-trans/facturas-trans.md)).

---

## Roles y acceso

| Acceso | Descripción |
|---|---|
| Autenticado | Requiere login |
| Rol específico | Requiere rol: `controller` |

---

## Estructura de archivos

```
importacion-facturas-envia/
├── importacion-facturas-envia.component.ts
├── importacion-facturas-envia.component.html
└── importacion-facturas-envia.component.scss
```

---

## Inputs y Outputs

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `isOpen` | `boolean` | `true` | Controla si el modal está visible. Al cambiar a `true` descarga la plantilla y carga la tabla |
| `closeEvent` | `EventEmitter<void>` | — | Emitido cuando el modal se cierra (por el usuario o al finalizar la inserción) |
| `refreshDataEvent` | `EventEmitter<void>` | — | Declarado para que el padre recargue sus datos. No se emite actualmente |

---

## Propiedades internas

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `form` | `FormGroup` | — | Formulario reactivo del modal |
| `isLoading` | `boolean` | `true` | Muestra el estado de carga durante la descarga de plantilla, validación o inserción |
| `fileToUpload` | `File \| null` | `null` | Archivo Excel seleccionado por el usuario |
| `uploadProgress` | `number` | `0` | Porcentaje de progreso de inserción (0–100) |
| `currentBatch` | `number` | `0` | Número de lote que se está procesando actualmente |
| `totalBatches` | `number` | `0` | Total de lotes calculados según el tamaño del archivo |
| `fileName` | `string` | `''` | Nombre de la plantilla (`Plantilla_facturas_envia.xlsx`) |
| `fileBlob` | `Blob` | — | Contenido binario de la plantilla para permitir su descarga |
| `rowsMemory` | `TablaRow[]` | `[]` | Registros ya existentes en `FacturacionEnvia`; se usan para calcular `IdCargaFactura` |

---

## Métodos

### `ngOnChanges(changes: SimpleChanges): void`

**Descripción:** Detecta cuando `isOpen` cambia a `true` y carga la plantilla y los registros existentes.

**Proceso:**
1. Detecta cambio en `isOpen`
2. Si el nuevo valor es `true`, llama a `obtenerArchivo()` y `fetchTableData()`

---

### `obtenerArchivo(): Promise<void>`

**Descripción:** Descarga la plantilla `Plantilla_facturas_envia.xlsx` desde el bucket S3 `logigho-plantillas`.

**Proceso:**
1. Llama a `GetObjectService.obtenerObjeto()` con bucket `logigho-plantillas`
2. Convierte la respuesta base64 a `Blob` con `base64ToBlob()`
3. Guarda el blob en `fileBlob` y el nombre en `fileName`
4. Si falla, muestra una advertencia `Error al cargar la plantilla`

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

**Descripción:** Procesa el archivo Excel subido por el usuario, **valida sus encabezados** y lanza la inserción en MongoDB.

**Proceso:**
1. Valida que existe un archivo seleccionado
2. Lee el archivo con `FileReader` en modo binario
3. Parsea el Excel con XLSX y extrae encabezados (primera fila) y filas
4. Define `columnMapping` y la función `mapColumns`
5. Ejecuta la [Validación 1](#validacion-de-encabezados) (primera fila de encabezados)
6. Ejecuta la [Validación 2](#validacion-de-encabezados) (nombres de columnas y obligatorios)
7. Mapea los encabezados con `mapColumns()` y filtra filas vacías
8. Fuerza a string el campo `Guia` para preservar ceros iniciales
9. Agrega `IdCargaFactura` (último id + 1) a cada registro
10. Llama a `procesarInsercionPorLotes()` con la colección `FacturacionEnvia`

---

### `fetchTableData(): Promise<void>` *(privado)*

**Descripción:** Consulta la colección `FacturacionEnvia` por páginas (grupos de 8 en paralelo), descomprime los datos y los deja en `rowsMemory`. Se usa para calcular el siguiente `IdCargaFactura`.

---

### `procesarInsercionPorLotes(registros: any[], coleccion: string): Promise<void>` *(privado)*

**Descripción:** Inserta los registros en MongoDB en lotes de 5000 de forma recursiva y secuencial, actualizando el progreso en cada lote.

**Parámetros:**
- `registros` — Array completo de objetos a insertar
- `coleccion` — Endpoint con la colección destino (`metodoGenerico?coleccion=FacturacionEnvia`)

**Proceso:**
1. Calcula el total de lotes (`Math.ceil(registros.length / 5000)`)
2. Por cada lote: llama a `insertarGenerico()`, actualiza `uploadProgress` y llama al siguiente lote
3. Al terminar: muestra alerta de éxito, recarga la tabla, resetea el input y cierra el modal
4. En caso de error: muestra alerta, resetea el progreso y el input

---

### Otros métodos

| Método | Descripción |
|---|---|
| `close()` | Cierra el modal, resetea el formulario y emite `closeEvent` |
| `resetFileInput()` | Limpia el valor del input de archivo para poder subir el mismo archivo otra vez |
| `base64ToBlob()` *(privado)* | Convierte el base64 recibido del API a `Blob` |
| `guessMimeTypeByFilename()` *(privado)* | Devuelve el tipo MIME según la extensión del archivo |
| `generateColumns()`, `capitalizeToPascalCase()` | Generan las columnas dinámicas de la tabla a partir de los registros |

---

## Validación de encabezados

Se ejecuta en `uploadFile()`, **antes** de mapear las columnas. Si cualquiera de las dos validaciones falla, el componente: quita el spinner (`isLoading = false`), limpia el archivo seleccionado (`resetFileInput()`), muestra una alerta con `Swal` y **detiene el proceso: no se inserta nada en MongoDB**. Además deja un `console.warn` con lo que falló (visible al inspeccionar la consola).

| # | Validación | Qué comprueba | Alerta si falla |
|---|---|---|---|
| 1 | Primera fila de encabezados | Todas las celdas de la primera fila deben ser texto y no estar vacías | `Archivo sin encabezados` |
| 2 | Nombres y obligatorios | Cada encabezado debe existir en `columnMapping` y deben venir todas las columnas obligatorias | `Encabezados incorrectos`, con las listas **No reconocidos** y **Faltan obligatorios** |

### Columnas de la plantilla

| Tipo | Columnas |
|---|---|
| **Obligatorias (11)** | `Numero_Factura`, `Guia`, `Reg_Origen`, `Ciu_Dest`, `Depto`, `Fec_Captura`, `Valor_Declarado`, `Valor_Flete`, `Flete_Variable`, `Valor_Otros`, `Total` |
| **No obligatorias (7)** | `Nom_Remitente`, `Nom_Destinatario`, `Num_Documentos`, `Dice_Contener`, `Texto_Guia`, `Accion_NotaGuia`, `Ctro_Costo` |

La lista de obligatorias es la constante `columnasObligatorias` dentro de `uploadFile()`. Si se necesitan más o menos campos obligatorios, se ajusta ahí.

### Por qué existe

Antes, un Excel sin fila de encabezados (la primera fila eran datos con números) hacía fallar la importación con el error `toLowerCase is not a function`, porque el código trataba esa fila como títulos. Además, un encabezado con nombre equivocado se guardaba en MongoDB como un campo nuevo, sin avisar.

### Código

```typescript
/**Validación 1: la primera fila debe ser de encabezados si trae números o celdas vacías
 *  se dispara el aviso y informa del error */
const headersArray: any[] = Array.from(headers);
const hayHeaderInvalido = headersArray.length === 0 ||
  headersArray.some(h => typeof h !== 'string' || h.trim() === '');
if (hayHeaderInvalido) {
  console.warn('Validación de encabezados falló: la primera fila no es de encabezados', headersArray);
  this.isLoading = false;
  this.resetFileInput();
  Swal.fire(
    'Archivo sin encabezados',
    'La primera fila del archivo debe tener los nombres de las columnas (texto, sin celdas vacías). Descarga la plantilla y copia tus datos debajo de esa fila.',
    'error'
  );
  return;
}

/**Validación 2: los nombres de los encabezados deben existir en columnMapping y deben venir los obligatorios.
 * Si hay nombres desconocidos o falta algún obligatorio, se dispara el aviso y informa del error*/
const columnasObligatorias = [
  "Numero_Factura", "Guia", "Reg_Origen", "Ciu_Dest", "Depto", "Fec_Captura",
  "Valor_Declarado", "Valor_Flete", "Flete_Variable", "Valor_Otros", "Total"
];
const nombresValidos = Object.keys(columnMapping).map(k => columnMapping[k].toLowerCase());
const nombresEnArchivo = headersArray.map(h => (h as string).toLowerCase());
// Encabezados del Excel que no están en columnMapping
const noReconocidos = headersArray.filter(h => !nombresValidos.includes((h as string).toLowerCase()));
// Obligatorios que no vinieron en el Excel
const faltantes = columnasObligatorias.filter(c => !nombresEnArchivo.includes(c.toLowerCase()));
if (noReconocidos.length > 0 || faltantes.length > 0) {
  console.warn('Validación de encabezados falló. No reconocidos:', noReconocidos, '| Faltantes:', faltantes);
  this.isLoading = false;
  this.resetFileInput();
  let detalle = 'Los nombres de las columnas no coinciden con la plantilla.';
  if (noReconocidos.length > 0) {
    detalle += `<br><br><b>No reconocidos:</b> ${noReconocidos.map(h => `"${h}"`).join(', ')}`;
  }
  if (faltantes.length > 0) {
    detalle += `<br><br><b>Faltan obligatorios:</b> ${faltantes.join(', ')}`;
  }
  Swal.fire({ title: 'Encabezados incorrectos', html: detalle, icon: 'error' });
  return;
}
```

---

## Flujo principal

```
Usuario abre modal (isOpen = true)
  → ngOnChanges()
    → obtenerArchivo()     → descarga plantilla desde S3
    → fetchTableData()     → carga registros existentes (para IdCargaFactura)

Usuario selecciona archivo
  → onFileChangeCarga()    → guarda archivo en fileToUpload

Usuario hace click en "Cargar"
  → uploadFile()
    → FileReader.readAsBinaryString()
      → XLSX.read()        → parsea el Excel
      → Validación 1       → ¿primera fila de encabezados?  (si falla: alerta y se detiene)
      → Validación 2       → ¿nombres válidos y obligatorios? (si falla: alerta y se detiene)
      → mapear columnas    → mapColumns()
      → filtrar vacíos
      → procesarInsercionPorLotes()
          → lote 1 → insertarGenerico() → uploadProgress 33%
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
| `ConsumoGenericoService` | `consultarGenerico()`, `insertarGenerico()` | Lectura e inserción de registros en MongoDB |
| `GetObjectService` | `obtenerObjeto()` | Descarga de la plantilla Excel desde S3 |
| `DecompressionService` | `decompressGzip()` | Descompresión de los datos paginados del API |
| `TiendasService` | `tiendas$` | Lista de tiendas (heredado, no se usa en la importación) |
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

```html
<!-- En el .html del padre (facturas-trans) -->
<app-importacion-facturas-envia
  [isOpen]="isEnviaModalOpen"
  (closeEvent)="closeEnviaModal()">
</app-importacion-facturas-envia>
```

---

## Changelog del componente

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-10-02 | Yadilton Torres | Validación de encabezados en `uploadFile()` (Validación 1: primera fila de encabezados; Validación 2: nombres existentes en `columnMapping` y columnas obligatorias). Corrige el error `toLowerCase is not a function` al subir un Excel sin fila de encabezados. |
| 2026-10-02 | Yadilton Torres | Eliminado un bloque de código comentado de manejo de fechas (`ADM_FechaGrabacion`): estaba copiado de otro módulo y nunca se ejecutaba. |

---

## Observaciones

- **Cambio de comportamiento:** un archivo con columnas fuera de `columnMapping` ahora se rechaza. Antes se aceptaba y se guardaba con el nombre que trajera el Excel. Probar con archivos reales de Envía.
- El componente asume que la plantilla `Plantilla_facturas_envia.xlsx` existe en el bucket S3 `logigho-plantillas`.
- El tamaño de lote está fijado en 5000 registros (`TAMANO_LOTE = 5000`).
- `Fec_Captura` se guarda tal como viene en el Excel; este componente no transforma fechas.
- `refreshDataEvent` está declarado pero no se emite. El padre recarga los datos con el evento `(updated)` de `app-tables`.
