# Módulo: Relación de Despacho

---

## Autor: Claude Code
Fecha creación: 2026-07-27
Estado: desarrollo
Tipo: módulo (1 vista + 1 componente hijo + 1 componente genérico compartido)

---

## Índice

1. [Vista: RelacionDespachoComponent](#1-vista-relaciondespachocomponent)
2. [Componente: CargaMasivaComponent](components/carga-masiva.md)
3. [Diagrama de flujo](relacion-despacho-flujo.md)

---

## 1. Vista: RelacionDespachoComponent

**Selector:** `app-relacion-despacho`
**Ubicación:** `src/app/views/logistica/relacion-despacho/relacion-despacho.component.ts`
**Acceso:** Logística → Relación de Despacho

---

### ¿Qué hace? (para el usuario)

Pantalla para registrar y organizar las guías que se van a despachar en el día. Al abrirla, se carga el borrador de guías pendientes (aún no incluidas en un PDF) y permite:

- **Registrar guías una por una** con lector de código de barras (USB) o digitándolas, o con la **cámara del celular** en mobile.
- **Cargar muchas guías a la vez** desde un archivo Excel/CSV (modal de carga masiva).
- **Buscar y filtrar** las guías por texto libre, tienda, transportadora o ecosistema.
- **Seleccionar** cuáles guías incluir al generar el PDF (o exportar todas las filtradas si no selecciona ninguna).
- **Generar el PDF** de la relación de despacho, agrupado por tienda y transportadora, y archivarlo en el histórico.
- **Consultar el histórico** de lotes ya despachados, filtrando por fecha y tienda, ver el detalle de un lote y volver a descargar o regenerar su PDF.
- **Eliminar** guías del borrador antes de despacharlas.

Incluye un tour guiado (`app-tour-guiado`) que resalta los controles principales la primera vez que se usa el módulo.

---

### Ruta

```
logistica/relacion-despacho
```

---

### Decoradores

```typescript
@Component({
  selector: 'app-relacion-despacho',
  standalone: true,
  imports: [CommonModule, FormsModule, CargaMasivaComponent, TourGuiadoComponent, ScannerOverlayComponent],
  templateUrl: './relacion-despacho.component.html',
  styleUrl: './relacion-despacho.component.scss',
})
export class RelacionDespachoComponent implements OnInit
```

Usa `@HostListener('document:click')` (`cerrarDropdowns()`) para cerrar todos los dropdowns de filtro al hacer clic fuera.

---

### Arquitectura: Store / Repository / Rules

El módulo separa estado, acceso a datos y reglas de negocio en tres capas, todas bajo `helpers/`:

| Capa | Archivo | Responsabilidad |
|---|---|---|
| `DespachoStore` | `helpers/despacho.store.ts` | `@Injectable({providedIn:'root'})`. Signals + computed. Única fuente de verdad compartida entre el pistolero (vista) y `CargaMasivaComponent` — ambos leen/escriben el mismo store inyectado. |
| `DespachoRepository` | `helpers/despacho.repository.ts` | Llamadas HTTP vía `ConsumoGenericoService` + `DecompressionService` (Zstd). El store nunca llama HTTP directo. |
| Reglas puras | `helpers/despacho.rules.ts` | Funciones sin estado ni HTTP: validación y construcción de registros. |

**Funciones clave de `despacho.rules.ts`:**

| Función | Qué hace |
|---|---|
| `validarTiendaAsignada(pedido, tiendasAsignadas)` | Rechaza si la tienda del pedido no está entre las asignadas al usuario (vacío = acceso a todas) |
| `validarEstadoGuia(pedido)` | Rechaza si `pedido.Estado` matchea contra la lista negra `ESTADOS_AVANZADOS` (tránsito, entregado, devuelto, cancelado, etc.) |
| `esDuplicado(guia, allRows)` | `true` si la guía ya está en el borrador, comparando por `guiaKey()` |
| `validarGuia(pedido, allRows, tiendasAsignadas)` | Orquesta las tres anteriores en orden: tienda → estado → duplicado |
| `guiaKey(guia)` | Normaliza a `string`, quitando ceros a la izquierda (`Number(guia)` + regex) — necesario porque `Guia` es `number` en Mongo y no conserva padding |
| `construirRegistro(pedido)` | Arma el `RegistroDespacho` a insertar, incluida `fechaColombia()` |
| `fechaColombia()` | `Date` actual menos 5 horas, en ISO — no hay conversión de zona horaria real, es un offset fijo |
| `esDeHoy(fechaIso)` | Compara el prefijo `YYYY-MM-DD` contra `fechaColombia()` |
| `generarIdLote(tienda)` | `DESP-<tiendaNormalizada>-<AAMMDD-HHmmss>` |
| `normalizarTienda(tienda)` | Quita tildes y caracteres no alfanuméricos (slug para `IdLote` y nombre de archivo del PDF) |

---

### Interfaz central del módulo

```typescript
// InventarioAdmision — borrador del día, aún no despachado
export interface RegistroDespacho {
  _id?: string;
  Guia: number;               // Number en Mongo, sin ceros a la izquierda
  NombreTienda: string;
  Transportadora: string;
  Cliente: string;
  Telefono?: number;
  Departamento?: string;
  TotalRecaudo: number;
  DiceContener?: string;
  Fecha: string;               // ISO, fechaColombia()
}

// HistorialDespachos — guía ya archivada
export interface GuiaDespachada extends RegistroDespacho {
  IdLote: string;
  FechaGeneracion: string;
  DespachoGenerado: boolean;   // heredado del módulo antiguo, se sigue guardando
}

// Fila del histórico agrupado (una por PDF/lote generado)
export interface ResumenLote {
  IdLote: string;
  NombreTienda: string;
  FechaGeneracion: string;
  TotalGuias: number;
  TotalRecaudo: number;
}

// Proyección mínima de PedidosInter que consume el módulo
export interface PedidoInter {
  _id: string;
  Numeropreenvio: string;
  Tienda: string;
  Transportadora: string;
  Telefono?: number;
  NombreCompleto: string;
  TotalRecaudo: number;
  DiceContener?: string;
  Departamento?: string;
  Estado?: string;
}

// Filtros activos sobre la tabla de guías pendientes
export interface DespachoFiltros {
  tiendas: string[];
  transportadoras: string[];
  busqueda: string;
}
```

> `Numeropreenvio` en `PedidoInter` sigue siendo `string` (Int64 de Mongo, riesgo de precisión); `RegistroDespacho.Guia` se persiste como `number`, consistente con el esquema real de `InventarioAdmision`.

---

### Colecciones Mongo

| Colección | Rol |
|---|---|
| `InventarioAdmision` | Borrador del día (guías aún no despachadas) |
| `PedidosInter` | Catálogo de pedidos. Colección core muy demandada: nunca se pide el documento completo, solo `CAMPOS_PEDIDO` proyectado |
| `HistorialDespachos` | Guías ya archivadas, con `IdLote`, `FechaGeneracion`, `DespachoGenerado: true` |
| `Tienda` | Mapa Ecosistema → Tiendas, para el filtro de ecosistema |

---

### Propiedades clave (vista)

| Propiedad | Tipo | Descripción |
|---|---|---|
| `readonly store` | `DespachoStore` | Inyectado como `public readonly` — la vista lee/escribe directo sobre sus signals desde el template |
| `activeTab` | `'pendientes' \| 'historico'` | Pestaña activa |
| `seleccionadas` | `RegistroDespacho[]` | Guías marcadas (checkbox) en "pendientes"; vacío = exportar/eliminar todas las filtradas |
| `seleccionadasLote` | `GuiaDespachada[]` | Selección dentro del detalle de un lote del histórico |
| `showCargaMasiva` | `boolean` | Abre/cierra el modal de carga masiva |
| `scannerAbierto` | `boolean` | Abre/cierra `<app-scanner-overlay>` (escaneo por cámara, mobile) |
| `tourAbierto` | `boolean` | Abre/cierra el tour guiado |
| `loteSeleccionado` | `string \| null` | `IdLote` cuyo detalle se está viendo, o `null` si se muestra la lista de lotes |
| `guiasLoteSeleccionado` | `GuiaDespachada[]` | Guías del lote abierto, cargadas vía `store.getGuiasDeLote()` |
| `regenerandoIdLote` | `string \| null` | `IdLote` cuyo PDF se está regenerando (deshabilita el botón de esa fila) |

**Propiedades clave (`DespachoStore`, signals):**

| Signal | Tipo | Descripción |
|---|---|---|
| `_allRows` | `RegistroDespacho[]` | Todo el borrador traído del backend |
| `_pedidosCache` | `Map<string, PedidoInter>` | Cache de `PedidosInter` indexado por `guiaKey()` |
| `_pedidosCacheListos` | `boolean` | `true` cuando terminó el prefetch inicial |
| `_filtros` | `DespachoFiltros` | Filtros activos sobre "pendientes" |
| `_lotes` | `ResumenLote[]` | Lotes del histórico cargados |
| `_fechaHistorico` | `string \| null` | Día filtrado en el histórico (`YYYY-MM-DD`) |
| `_tiendasHistorico` | `string[]` | Filtro multi-select client-side sobre lotes ya cargados |
| `_idsPendientesDeLimpiar` | `number[]` | Guías del borrador que no se pudieron eliminar tras un `exportarLote` parcial |

**Computed clave:**

| Computed | Lógica |
|---|---|
| `rows` | `_allRows` filtrado por `esDeHoy(r.Fecha)` |
| `rowsFiltrados` | `rows` + filtros de tienda/transportadora (OR interno, AND entre categorías) + búsqueda de texto libre (guía, cliente, tienda) |
| `rowsPagina` | Slice de `rowsFiltrados` según `pagina()`, `PAGE_SIZE = 10` |
| `tiendasDisponibles` / `transportadorasDisponibles` | Únicas presentes en `rows()` (no en todo el borrador) |
| `ecosistemasDisponibles` | Ecosistemas de `_tiendaMap` que tienen al menos una tienda en `tiendasDisponibles()` |
| `lotesFiltrados` | `_lotes` filtrado por `_tiendasHistorico` (OR) |

---

### Servicios y endpoints

| Servicio | Uso |
|---|---|
| `ConsumoGenericoService` | GET/POST/DELETE al backend genérico |
| `DecompressionService` | Descomprime respuestas Zstd |
| `DespachoRepository` | Encapsula las llamadas anteriores; el store solo habla con él |

**Patrón de URL:** `metodoGenerico?coleccion=<COLECCION>&<filtros>&mcomp=2`. El filtro de tienda se omite si `sessionStorage.tiendas_asignadas` incluye `"Todas"`.

| Colección | Operación | Cuándo |
|---|---|---|
| `InventarioAdmision` | GET (filtrado por tienda) | `iniciar()`, `refrescarInventario()` |
| `InventarioAdmision` | POST | `procesarGuia()`, `confirmarLote()` |
| `InventarioAdmision` | DELETE (`Guia=a,b,c`, chunks de 300) | `eliminarVariosDelBorrador()`, `exportarLote()`, `reintentarLimpieza()` |
| `PedidosInter` | GET proyectado (`campos=Numeropreenvio,Tienda,Transportadora,...`) | `getPedidosPrefetch()` (sin pipeline agregado — se quitó por lentitud en API Gateway) |
| `PedidosInter` | GET por guía única | `buscarPedidoPorGuia()`, fallback del pistolero si no está en cache |
| `PedidosInter` | GET `$in` (`Numeropreenvio=a,b,c`) | `buscarPedidosPorGuias()`, batch de 300 para carga masiva |
| `Tienda` | GET | `getMapaTiendas()` → Ecosistema → [Tiendas] |
| `HistorialDespachos` | POST | `archivarLote()` |
| `HistorialDespachos` | GET con `pipeline` (`$group` por `IdLote`, base64 de stages) | `getResumenLotes()` |
| `HistorialDespachos` | GET (`IdLote=...`) | `getGuiasDeLote()` |

---

### Flujo principal

```
ngOnInit() → store.iniciar()
  ├─► getInventario()          → _allRows
  ├─► getPedidosPrefetch()     → _pedidosCache (paralelo, no bloquea)
  └─► getMapaTiendas()         → _tiendaMap (paralelo)

Pistolero / escáner cámara → onGuiaEnter() → store.procesarGuia(guia)
  ├─► obtenerPedido()   [cache-first, fallback buscarPedidoPorGuia]
  ├─► validarGuia()     [tienda → estado → duplicado]
  ├─► inserción optimista en _allRows
  └─► insertarRegistros() [POST]

Carga masiva → store.validarLote(guias) → store.confirmarLote(validos)
  ├─► resuelve cache-first, batch de 300 (buscarPedidosPorGuias) para lo faltante
  ├─► validarGuia() por cada guía
  └─► confirmarLote() → insertarRegistros() + refrescarInventario()

Exportar PDF (pestaña "pendientes") → exportToPDF()
  ├─► fuente = seleccionadas.length ? seleccionadas : rowsFiltrados
  ├─► bloquea si hay >1 tienda mezclada y no hay filtro de tienda explícito
  ├─► agrupa por NombreTienda → chunks de MAX_GUIAS_POR_PDF = 300
  └─► por chunk: generarPdfDespacho() → store.exportarLote(chunk, tienda)
        ├─► POST a HistorialDespachos (IdLote, FechaGeneracion, DespachoGenerado:true)
        ├─► DELETE tolerante del borrador
        └─► si falla la limpieza → idsPendientesDeLimpiar + aviso de reintento

Pestaña "Histórico" → cambiarTab('historico') → store.cargarLotes()
  ├─► getResumenLotes(fecha) [pipeline $group sobre HistorialDespachos]
  ├─► filtro de tienda client-side (setTiendasHistorico)
  └─► abrirDetalleLote(idLote) → getGuiasDeLote(idLote)
        └─► exportarSubsetDeLote() [genera PDF, no archiva de nuevo]
        └─► regenerarPdf(idLote) [en la lista, sin abrir detalle: solo regenera]
```

---

### Bloqueo de export mezclado

Si la fuente a exportar contiene guías de más de una tienda (`grupos.size > 1`) y el usuario no aplicó ningún filtro de tienda (`tiendasFiltro.length === 0`), `exportToPDF()` bloquea la operación con una alerta. Esto evita que un usuario con acceso a "Todas" las tiendas archive/exporte por error guías de una tienda que no es la que pretendía despachar. Si el usuario sí seleccionó tiendas en el multi-select, el export continúa generando un PDF y un `IdLote` separado por cada tienda.

---

### Escaneo por cámara (mobile)

El botón `rd-scan-btn` solo es visible en `@media (max-width: 768px)`. Abre `<app-scanner-overlay>` (usa `@zxing/ngx-scanner`), un componente genérico y reutilizable (`src/app/components/scanner-overlay/`) que emite `(scanSuccess)` con el valor leído; ese evento se conecta directo a `onGuiaEnter()`, el mismo flujo que el pistolero de texto/USB.

---

### Tour guiado

`app-tour-guiado` es un componente genérico (`src/app/components/tour-guiado/`) reutilizado en varios módulos. Este módulo define su propio array `tourSteps` apuntando a los ids: `rd-pistolero`, `rd-scan-btn`, `rd-filtros`, `rd-btn-masiva`, `rd-tabla`, `rd-btn-pdf`, `rd-tabs`.

---

### Responsive / mobile

- La toolbar de filtros se apila verticalmente en `@media (max-width: 768px)`.
- Los dropdowns (tienda, transportadora, ecosistema, fecha) pasan de panel flotante a **bottom-sheet fijo con backdrop** en mobile.
- `z-index` de los paneles subido a `20100` / `20050` para quedar por encima del sidebar del layout general (`z-index: 20000`).

---

### Historial de cambios

| Fecha | Cambio |
|---|---|
| 2026-07-03 | Creación del módulo original: pistolero, tabla, PDF agrupado por transportadora (arquitectura reemplazada) |
| 2026-07-27 | Reescritura sobre arquitectura Store/Repository/Rules, carga masiva vía Excel, cache-first de `PedidosInter`, escáner por cámara, pestaña de histórico agrupado por lote con detalle y regeneración de PDF |

---

### Observaciones

- `getPedidosPrefetch()` no usa pipeline agregado: se quitó porque generaba lentitud en el API Gateway. En su lugar, trae `PedidosInter` proyectado a campos mínimos y cachea en memoria indexado por `guiaKey()`.
- El PDF se genera con `jsPDF` + `html2canvas` + `JsBarcode`, renderizando el HTML dentro de un `<iframe>` oculto fuera de la ventana visible para aislarlo de estilos globales; se trocea el canvas en páginas A4.
- El PDF se genera **antes** de archivar en `HistorialDespachos`: si `generarPdfDespacho()` falla, ese chunk nunca se mueve del borrador.
- Si `exportarLote()` archiva correctamente pero el `DELETE` del borrador falla parcialmente, las guías quedan en `idsPendientesDeLimpiar()` y el usuario puede reintentar la limpieza sin volver a generar el PDF ni duplicar el archivado.
- El detalle de un lote del histórico reutiliza la estructura de tabla/columnas/checkboxes de "pendientes", pero con datos y paginación **locales al componente** (no viven en el store).
