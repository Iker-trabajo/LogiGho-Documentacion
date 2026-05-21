---
autor: Adalberto González
fecha_creacion: 2026-04-24
ultima_actualizacion: 2026-05-06
estado: Desarrollo
nivel: 2
---

# Vista: Pauta

**Autor:** Adalberto González  

**Selector:** `app-pauta`  

**Ubicación:** `SitioLogiGho/src/app/views/tienda/pauta`

---

## ¿Qué hace?

Vista de gestión de inversión publicitaria por tienda. Permite listar, buscar, editar inline y descargar los registros de pauta (Meta, TikTok, Google y Otros) filtrados por las tiendas asignadas al usuario. También permite importar nuevos registros desde un archivo Excel y descargar la plantilla de importación desde S3.

---

## Ruta

| Propiedad | Valor |
|---|---|
| **Ruta** | `/tienda/pauta` |
| **Título de página** | `Pautas` |
| **Guard** | `AuthGuard` |
| **Rol requerido** | Todos los usuarios autenticados |
| **Parámetros de URL** | Ninguno |

---

## Estructura de archivos

```
pauta/
├── pauta.component.ts
├── pauta.component.html
└── pauta.component.scss
```

---

## Secciones de la vista

| # | Sección | Descripción |
|---|---|---|
| 1 | **Cabecera** | Título "Pautas", subtítulo y tres botones: `Descargar Plantilla`, `Descargar Registros` e `Importar` |
| 2 | **Modal de importación** | `app-importacion-pauta` — se abre con el botón Importar |
| 3 | **Tarjetas de resumen** | 4 stat-cards: Total Meta, Total TikTok, Total Google y Otros (suman el conjunto de datos filtrado activo) |
| 4 | **Barra de búsqueda** | Input de texto con debounce de 220 ms y contador de registros filtrados |
| 5 | **Tabla de pautas** | Listado paginado con edición inline por fila: Fecha, ID Tienda, Tienda, Meta, TikTok, Google, Otro y Acción |
| 6 | **Pie de tabla / Paginación** | Selector de registros por página (5, 10, 25, 50) y controles de paginación con elipsis |

---

## Propiedades del componente

### Datos

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `pauta` | `any[]` | `[]` | Lista activa (puede estar filtrada). La tabla y las stat-cards operan sobre esta lista. |
| `pautaOriginal` | `any[]` | `[]` | Copia inmutable de los datos cargados desde BD. Fuente de verdad para el filtro. |
| `modoEdicion` | `boolean[]` | `[]` | Array indexado por posición en `pauta`. `true` si la fila está en modo edición. |
| `registroBackup` | `any` | `{}` | Objeto clave-índice que guarda una copia del registro antes de editar, para poder cancelar. |

### Búsqueda y filtro

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `filtro` | `string` | `''` | Texto del buscador. Se evalúa contra todos los campos del registro. |
| `filterTimeout` | `number \| null` | `null` | ID del `setTimeout` de debounce. Se cancela en cada keystroke antes de crear uno nuevo. |

### Caché de totales

| Propiedad | Tipo | Descripción |
|---|---|---|
| `totalsCacheVersion` | `number` | Versión del caché. Se incrementa al filtrar o guardar para forzar recálculo. |
| `totalsCache` | `Record<string, number>` | Almacena totales calculados, indexados por `campo_versión`. Evita recalcular en cada ciclo de detección de cambios. |

### Paginación

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `currentPage` | `number` | `1` | Página actual. |
| `pageSize` | `number` | `10` | Registros por página. |
| `pageSizeOptions` | `number[]` | `[5,10,25,50]` | Opciones del selector de tamaño de página. |
| `totalPaginas` | `number` | `0` | Total de páginas calculado. |
| `paginasVisibles` | `number[]` | `[]` | Array de números de página a renderizar. `-1` representa `...`. |

### Modal

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `isImpPautaModalOpen` | `boolean` | `false` | Controla la visibilidad del modal de importación. |

---

## Interfaz de datos (implícita)

```typescript
interface PautaRegistro {
  _id: string;          // ObjectId de MongoDB
  Fecha: string;        // Fecha del registro (ej. "2026-04-01")
  Idtienda: number;     // ID numérico de la tienda
  Tienda: string;       // Nombre de la tienda
  Meta: number | null;  // Inversión en Meta Ads
  TikTok: number | null;
  Google: number | null;
  Otro: number | null;
}
```

---

## Flujo de inicialización

```
ngOnInit()
  → fetchTableData()
      → Lee tiendas_asignadas de sessionStorage
      → GET metodoGenerico?coleccion=Pauta&Tienda=...
      → decompressGzip(data.Resultado)
      → Mapea resultado a PautaRegistro[]
      → this.pauta = resultado
      → this.pautaOriginal = [...this.pauta]
      → Inicializa modoEdicion[]
      → invalidateTotalsCache()
      → calcularPaginacion()
```

---

## Métodos

### `fetchTableData(): Promise<any[]>`
Carga los registros de pauta desde la BD, filtrados por las tiendas asignadas al usuario.

**Proceso:**
1. Lee `tiendas_asignadas` de `sessionStorage`.
2. Consulta `metodoGenerico?coleccion=Pauta&Tienda=...`.
3. Descomprime con `decompressGzip`.
4. Mapea y asigna a `pauta` y `pautaOriginal`.
5. Inicializa `modoEdicion` con `false` para cada registro.
6. Invalida el caché de totales y recalcula la paginación.

---

### `filtrarTabla()`
Aplica un debounce de 220 ms antes de llamar a `applyFilter()`. Cancela el timeout anterior en cada keystroke para no ejecutar filtros intermedios.

---

### `applyFilter()` *(privado)*
Filtra `pautaOriginal` comparando el texto de `filtro` contra todos los campos del registro (mediante `Object.values`). Resetea la página a 1, invalida el caché y recalcula la paginación.

---

### `editarFila(index)`
Habilita el modo edición para la fila en `index`.

**Proceso:**
1. Crea una copia del registro en `registroBackup[index]`.
2. Pone `modoEdicion[index]` en `true`.

---

### `guardarFila(index): Promise<void>`
Persiste los cambios de la fila en edición en la BD.

**Proceso:**
1. Valida que `Fecha`, `Tienda` e `Idtienda` no estén vacíos.
2. Convierte `Idtienda`, `Meta`, `TikTok`, `Google` y `Otro` a `number`.
3. Valida que los campos numéricos no sean `NaN`.
4. Verifica que el registro tenga `_id`.
5. Llama a `actualizarGenerico(registro, registro._id, 'metodoGenerico?coleccion=Pauta')`.
6. Pone `modoEdicion[index]` en `false`.
7. Invalida el caché de totales.

**Manejo de error:** `Swal.fire` de error si la llamada falla.

---

### `cancelarEdicion(index)`
Descarta los cambios de la fila restaurando el registro desde `registroBackup[index]` y deshabilita el modo edición.

---

### `calcularTotal(campo): number`
Suma todos los valores del `campo` en `this.pauta` (datos filtrados activos).

**Optimización:** Usa `totalsCache` indexado por `campo_version`. Solo recalcula cuando `totalsCacheVersion` cambia (al filtrar o guardar). Evita recalcular en cada ciclo de detección de cambios de Angular.

---

### `onPlantillaClick()`
Descarga el archivo `Pauta.xlsx` desde el bucket S3 `logigho-plantillas` vía `GetObjectService`. Convierte la respuesta base64 a `Blob` y lo descarga mediante un enlace temporal.

---

### `descargarPauta()`
Exporta los registros visibles (`this.pauta`) a un archivo `Registro_Pautas.xlsx` usando `XLSX`. Agrega una fila de totales al final del listado.

---

### `trackByPauta(index, item)`
Función `trackBy` para `*ngFor`. Retorna `item._id` si existe, o `index` como fallback. Evita re-renderizar filas que no cambiaron.

---

### Paginación

| Método | Descripción |
|---|---|
| `pautaPaginada` (getter) | Slice de `pauta` para la página actual. |
| `cambiarPagina(page)` | Cambia `currentPage` si `page` está en rango y recalcula páginas visibles. |
| `cambiarSizePagina(event)` | Actualiza `pageSize`, resetea a página 1 y recalcula. |
| `calcularPaginacion()` | Calcula `totalPaginas` y llama a `calcularPaginasVisibles()`. |
| `calcularPaginasVisibles()` | Genera `paginasVisibles` con lógica de elipsis: muestra todas si ≤ 7 páginas; agrega `−1` (`...`) para inicio, medio o final. |

---

### `openImportInveModal()` / `closeImpInveModal()`
Controlan `isImpPautaModalOpen` para abrir y cerrar el modal `app-importacion-pauta`.

---

## Subcomponentes

| Componente | Selector | Cuándo se monta | Evento de retorno |
|---|---|---|---|
| `ImportacionPautaComponent` | `app-importacion-pauta` | Siempre (controlado por `[isOpen]`) | `(closeEvent)` → `closeImpInveModal()` / `(refreshDataEvent)` → `fetchTableData()` |

---

## Servicios utilizados

| Servicio | Métodos usados | Propósito |
|---|---|---|
| `ConsumoGenericoService` | `consultarGenerico()`, `actualizarGenerico()` | Carga y actualización de registros de pauta |
| `DecompressionService` | `decompressGzip()` | Descomprime las respuestas del backend |
| `GetObjectService` | `obtenerObjeto()` | Descarga la plantilla Excel desde S3 |
| `Router` | — | Inyectado pero sin uso activo |
| `ActivatedRoute` | — | Inyectado pero sin uso activo |

---

## Endpoints que consume

| Método | Ruta | Cuándo |
|---|---|---|
| `GET` | `metodoGenerico?coleccion=Pauta&Tienda={tiendas}` | Al inicializar y tras importar |
| `PUT/PATCH` | `metodoGenerico?coleccion=Pauta` (vía `actualizarGenerico`) | Al guardar una fila editada |
| `GET` | S3 bucket `logigho-plantillas`, key `Pauta.xlsx` | Al descargar la plantilla |

---

## Estados de la vista

| Estado | Descripción | Qué muestra |
|---|---|---|
| Cargando | `fetchTableData()` en curso | Tabla vacía (sin spinner explícito) |
| Con datos | Registros cargados | Tabla con paginación y stat-cards |
| Filtrado activo | Texto en el buscador | Tabla reducida; stat-cards reflejan solo los filtrados |
| Fila en edición | `modoEdicion[i] = true` | Inputs inline reemplazan los spans; botones Guardar/Cancelar |
| Sin resultados | Filtro no coincide | Fila `"No se encontraron registros de pauta."` |
| Error de carga | Falla la llamada al backend | `Swal.fire` de error |

---

## Changelog de la vista

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-04-24 | Adalberto González | Personalizacion de estilos en el scss de el componente |
| 2026-04-24 | Adalberto González | Peronalizacion de la tabla que muestra los datos, ahora podemos filtrar y paginar |
| 2026-05-16 | Adalberto González | Se volvio a añadir el editar fecha en la tabla de el modulo |

---

## Observaciones

- **`Router` y `ActivatedRoute` sin uso:** Inyectados en el constructor pero sin llamadas activas. Pueden eliminarse.
- **`productosFiltrados`, `isLoading`, `isImage`, `isPdf`, `fileToUpload`, `isHovered`, `fileBlob`, `fileUrl`, `extensionArchivo`, `fileName`:** Propiedades declaradas pero sin uso activo en la vista. Son residuo de un template anterior y pueden eliminarse para reducir ruido.
- **Sin confirmación antes de guardar:** `guardarFila()` persiste directamente sin `Swal.fire` de confirmación previo, a diferencia del flujo de eliminación en `PortafolioComponent`.
- **Edición inline limitada a la vista paginada:** `modoEdicion` está indexado por posición en `pautaPaginada`, no por `_id`. Si el usuario cambia de página con una fila en edición, el índice puede desfasarse con respecto a `pauta`. Se recomienda indexar por `_id`.
