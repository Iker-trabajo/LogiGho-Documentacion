## Componente: CargaMasivaComponent

**Selector:** `app-carga-masiva`
**Ubicación:** `src/app/views/logistica/relacion-despacho/components/carga-masiva/carga-masiva.component.ts`
**Acceso:** Modal abierto desde el botón `rd-btn-masiva` de Relación de Despacho

---

### ¿Qué hace? (para el usuario)

Modal para registrar muchas guías de una sola vez, sin pistolerarlas una por una:

1. El usuario descarga (opcional) una plantilla Excel de ejemplo.
2. Arrastra o selecciona un archivo `.xlsx`, `.xls` o `.csv` con una columna **Guia**.
3. Al cargar, ve una barra de progreso con el paso actual.
4. Al terminar, ve un resumen: cuántas guías se registraron, cuántas se rechazaron y por qué motivo cada una.
5. Puede cargar otro archivo sin cerrar el modal.

Incluye su propio mini-tour guiado (`tourSteps` internos: `cm-instrucciones`, `cm-plantilla`, `cm-upload`, `cm-footer`).

---

### Decoradores y configuración técnica

```typescript
@Component({
  selector: 'app-carga-masiva',
  standalone: true,
  imports: [CommonModule, TourGuiadoComponent],
  templateUrl: './carga-masiva.component.html',
  styleUrl: './carga-masiva.component.scss',
})
export class CargaMasivaComponent
```

Usa `@HostListener('document:keydown.escape')` para cerrar el modal (no actúa mientras `isLoading` es `true`).

No hace HTTP directo ni duplica reglas de validación: toda la lógica de negocio se delega al `DespachoStore` inyectado, la misma fuente de verdad que usa el pistolero.

---

### Propiedades clave

**Inputs/Outputs:**

| Propiedad | Tipo | Descripción |
|---|---|---|
| `@Input() isOpen` | `boolean` | Controla la visibilidad del modal |
| `@Output() close` | `EventEmitter<void>` | Emitido al cerrar (botón ✕, backdrop o `Escape`) |

**Estado interno:**

| Propiedad | Tipo | Descripción |
|---|---|---|
| `selectedFile` | `File \| null` | Archivo seleccionado |
| `isLoading` | `boolean` | `true` mientras se lee/valida el archivo |
| `uploadProgress` | `number` | Porcentaje (0–100) de la barra de progreso |
| `currentStep` | `string` | Texto descriptivo del paso actual |
| `isDragOver` | `boolean` | `true` mientras se arrastra un archivo sobre la zona de carga |
| `showResults` | `boolean` | `true` al terminar, muestra la sección de resultados |
| `totalRecords` | `number` | Guías encontradas en el archivo |
| `successfulRecords` | `number` | Guías válidas insertadas |
| `rejectedRecords` | `number` | Guías rechazadas |
| `rejectedDetails` | `GuiaRechazada[]` | `{ guia, motivo }[]` para la tabla de resultados |

**Métodos:**

| Método | Descripción |
|---|---|
| `onFileChange(event)` / `onDrop(event)` | Toman el archivo del input nativo o del drag&drop |
| `processFile(file)` (privado) | Valida extensión (`xlsx`/`xls`/`csv`); rechaza con `Swal` si no coincide |
| `uploadFile()` | Lee el archivo con `FileReader` + `XLSX.read()`, valida que exista columna `Guia`/`Guía`, extrae la lista de guías y llama `procesarGuias()` |
| `procesarGuias(guias)` (privado) | `store.validarLote(guias)` → `store.confirmarLote(validos)` |
| `downloadTemplate()` | Genera y descarga `Plantilla_Despacho.xlsx` con `XLSX.utils.aoa_to_sheet()` |
| `loadAnotherFile()` | Vuelve a la pantalla de selección sin cerrar el modal |
| `resetForm()` / `resetFileInput()` | Restauran el estado inicial del componente |
| `getReasonClass(motivo)` | Clase CSS del badge de rechazo (`reason-info` si es "no encontrado(a)", si no `reason-warning`) |

---

### Flujo principal

```
Usuario selecciona/arrastra archivo
  └─► processFile() → valida extensión → selectedFile

Usuario hace clic en "Cargar Archivo"
  └─► uploadFile()
        ├─► FileReader.readAsBinaryString()
        ├─► XLSX.read() → sheet_to_json({ header: 1 })
        ├─► valida columna "Guia"/"Guía"
        ├─► extrae guías (columna 0, filas > 0, no vacías)
        └─► procesarGuias(guias)
              ├─► store.validarLote(guias)
              │     ├─► resuelve contra cache del prefetch (guiaKey)
              │     ├─► batch de 300 (buscarPedidosPorGuias) para lo faltante
              │     └─► valida cada guía con validarGuia() (tienda/estado/duplicado)
              └─► store.confirmarLote(validos)
                    ├─► insertarRegistros() [POST InventarioAdmision]
                    └─► refrescarInventario()

showResults = true → tabla de rechazados con motivo
```

---

### Historial de cambios

| Fecha | Cambio |
|---|---|
| 2026-07-27 | Reescritura sobre `DespachoStore.validarLote()` / `confirmarLote()`: la validación de negocio (tienda, estado, duplicado) se resuelve contra el cache de `PedidosInter` cache-first + batch de 300, en vez de un request por guía |

---

### Observaciones

- El componente nunca llama `ConsumoGenericoService` directamente; todo pasa por el `DespachoStore` inyectado, que es el mismo store que usa la vista para el pistolero — así ambos flujos comparten el borrador en memoria (`_allRows`) sin duplicar lógica de validación.
- `XLSX.read()` se llama con `{ type: 'binary' }` porque el archivo se lee con `readAsBinaryString()`, no `readAsArrayBuffer()`.
- El escape del teclado se ignora mientras `isLoading` es `true`, para no permitir cerrar el modal a mitad de una carga.
