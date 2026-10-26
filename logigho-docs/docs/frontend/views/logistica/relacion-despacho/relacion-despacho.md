# Módulo: Relación de Despacho

---

## Autor: Adalberto González
Fecha creación: 2026-07-27  
Última actualización: 2026-10-26  
Estado: desarrollo  
Tipo: módulo (1 vista + 1 componente hijo + 2 componentes genéricos compartidos)

---

## Índice

1. [Vista: RelacionDespachoComponent](#1-vista-relaciondespachocomponent)
2. [Histórico de lotes](#historico-de-lotes)
3. [Helpers (store, repository, rules, pdf)](#helpers)
4. [Componente: CargaMasivaComponent](components/carga-masiva.md)
5. [Diagrama de flujo](relacion-despacho-flujo.md)

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
- **Consultar el histórico** de lotes ya despachados: filtrar por fecha (un día o un rango), ecosistema, tienda y transportadora; buscar en qué lote quedó una o varias guías; ver el detalle de un lote; volver a descargar o regenerar su PDF; y **unir varios lotes** en un solo PDF.
- **Eliminar** guías del borrador antes de despacharlas.

Las guías de **Envía** se muestran siempre con su 0 inicial (en tablas, PDF y código de barras), igual que las imprime la transportadora.

Incluye un tour guiado (`app-tour-guiado`) de 11 pasos que recorre las dos pestañas.

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
  imports: [CommonModule, FormsModule, CargaMasivaComponent, TourGuiadoComponent, EscanerCamaraComponent],
  templateUrl: './relacion-despacho.component.html',
  styleUrls: [
    './styles/relacion-despacho.scss',
    './styles/relacion-despacho.mobile.scss',
  ],
})
export class RelacionDespachoComponent implements OnInit
```

- `@HostListener('document:click')` (`cerrarDropdowns()`): cierra todos los menús desplegables al hacer clic fuera.
- `@HostListener('document:keydown.escape')` (`onEscape()`): cierra primero el aviso "¿Unir lotes?" si está abierto; si no, cierra el detalle del lote (salvo con carga masiva, tour o escáner abiertos).

---

### Estructura de carpetas

```
relacion-despacho/
├── relacion-despacho.component.ts / .html / .spec.ts
├── styles/
│   ├── relacion-despacho.scss          # estilos de PC
│   └── relacion-despacho.mobile.scss   # tablet (≤860px), celular (≤768px) y ≤400px
├── components/carga-masiva/
├── helpers/
│   ├── despacho.store.ts / .spec.ts
│   ├── despacho.repository.ts / .spec.ts
│   ├── despacho.repository.fake.ts     # repositorio falso, solo para pruebas
│   ├── despacho.rules.ts / .spec.ts
│   └── despacho-pdf.helper.ts
└── models/despacho.models.ts
```

El archivo móvil se carga después del de PC, así que solo sobrescribe lo necesario por tamaño de pantalla.

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
| `guiaParaMostrar(guia, transportadora)` | Guía tal como la imprime la transportadora. Si es **Envía** (sin tildes, en mayúsculas, por `includes('ENVIA')`) y no empieza por 0, le agrega el 0 inicial. No cuenta dígitos. Se usa en tablas, tarjetas, PDF y código de barras |
| `construirRegistro(pedido)` | Arma el `RegistroDespacho` a insertar, incluida `fechaColombia()` |
| `fechaColombia()` | `Date` actual menos 5 horas, en ISO — no hay conversión de zona horaria real, es un offset fijo |
| `esDeHoy(fechaIso)` | Compara el prefijo `YYYY-MM-DD` contra `fechaColombia()` |
| `generarIdLote(tienda)` | `DESP-<tiendaNormalizada>-<AAMMDD-HHmmss>` |
| `normalizarTienda(tienda)` | Quita tildes y caracteres no alfanuméricos (slug para `IdLote` y nombre de archivo del PDF) |

> El 0 de Envía es **solo de presentación**: en `InventarioAdmision` y `HistorialDespachos` la guía sigue guardada como número, sin el 0. Las búsquedas usan `guiaKey()`, así que encuentran la guía con o sin el 0.

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
  Transportadoras: string[];   // distintas del lote (normalmente una sola)
}

// Rango de días del filtro del histórico (inclusivo). Un solo día: desde === hasta
export interface RangoFechas {
  desde: string;               // YYYY-MM-DD
  hasta: string;               // YYYY-MM-DD
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
| `readonly store` | `DespachoStore` | Inyectado como `readonly` — la vista lee/escribe directo sobre sus signals desde el template |
| `activeTab` | `'pendientes' \| 'historico'` | Pestaña activa |
| `seleccionadas` | `RegistroDespacho[]` | Guías marcadas (checkbox) en "pendientes"; vacío = exportar/eliminar todas las filtradas |
| `seleccionadosLotesHistorico` | `string[]` | `IdLote` marcados en la tabla del histórico, para unirlos en un PDF |
| `confirmandoUnion` | `boolean` | Muestra el aviso "¿Unir lotes?" antes de generar el PDF unido |
| `seleccionadasLote` | `GuiaDespachada[]` | Selección dentro del detalle de un lote del histórico |
| `showCargaMasiva` | `boolean` | Abre/cierra el modal de carga masiva |
| `scannerAbierto` | `boolean` | Abre/cierra `<app-escaner-camara>` (escaneo por cámara, mobile) |
| `tourAbierto` | `boolean` | Abre/cierra el tour guiado |
| `loteSeleccionado` | `string \| null` | `IdLote` cuyo detalle se está viendo, o `null` si se muestra la lista de lotes |
| `guiasLoteSeleccionado` | `GuiaDespachada[]` | Guías del lote abierto, cargadas vía `store.getGuiasDeLote()` |
| `pageSizeLoteDetalle` | `number` | Guías por página en el detalle del lote (10, 25, 50 o 100) |
| `regenerandoIdLote` | `string \| null` | `IdLote` cuyo PDF se está regenerando (deshabilita el botón de esa fila) |
| `modoBusquedaGuias` | `'lote' \| 'guia'` | Modo del buscador del histórico |
| `busquedaGuiasHistorico` | `string` | Texto del buscador (una guía o varias separadas por coma) |
| `guiasNoEncontradas` | `string[]` | Guías buscadas (modo Lote) que no están en ningún lote |
| `guiaDestacada` | `string \| null` | Guía buscada en modo Guía: el detalle del lote muestra solo esa |
| `dropdown*Abierto` | `boolean` | Un flag por menú (fecha, ecosistema, tienda, transportadora de cada pestaña y "Ver N" del histórico y del detalle). Solo uno abierto a la vez (`toggleDropdown()`) |

**Propiedades clave (`DespachoStore`, signals):**

| Signal | Tipo | Descripción |
|---|---|---|
| `_allRows` | `RegistroDespacho[]` | Todo el borrador traído del backend |
| `_pedidosCache` | `Map<string, PedidoInter>` | Cache de `PedidosInter` indexado por `guiaKey()` |
| `_pedidosCacheListos` | `boolean` | `true` cuando terminó el prefetch inicial |
| `_filtros` | `DespachoFiltros` | Filtros activos sobre "pendientes" |
| `_lotes` | `ResumenLote[]` | Lotes del histórico cargados según el filtro de fecha |
| `_fechaHistorico` | `RangoFechas \| null` | Día o rango filtrado en el histórico |
| `_tiendasHistorico` | `string[]` | Filtro multi-select client-side de tiendas sobre los lotes cargados |
| `_transportadorasHistorico` | `string[]` | Filtro multi-select client-side de transportadoras |
| `_lotesBusquedaGuias` | `ResumenLote[] \| null` | Lotes encontrados por el buscador de guías; `null` = no hay búsqueda activa |
| `_pageSizeHistorico` | `number` | Lotes por página (`OPCIONES_PAGE_SIZE_HISTORICO = [10, 25, 50, 100]`) |
| `_idsPendientesDeLimpiar` | `number[]` | Guías del borrador que no se pudieron eliminar tras un `exportarLote` parcial |

**Computed clave:**

| Computed | Lógica |
|---|---|
| `rows` | `_allRows` filtrado por `esDeHoy(r.Fecha)` |
| `rowsFiltrados` | `rows` + filtros de tienda/transportadora (OR interno, AND entre categorías) + búsqueda de texto libre (guía, cliente, tienda) |
| `rowsPagina` | Slice de `rowsFiltrados` según `pagina()`, `PAGE_SIZE = 10` |
| `tiendasDisponibles` / `transportadorasDisponibles` | Únicas presentes en `rows()` (no en todo el borrador) |
| `ecosistemasDisponibles` | Ecosistemas de `_tiendaMap` que tienen al menos una tienda en `tiendasDisponibles()` |
| `hayBusquedaGuias` | `true` si `_lotesBusquedaGuias` no es `null` |
| `lotesBase` (privado) | `_lotesBusquedaGuias ?? _lotes`: la búsqueda de guías reemplaza la lista mientras está activa |
| `lotesFiltrados` | `lotesBase` filtrado por tiendas (OR) y por transportadoras (el lote pasa si tiene alguna de las elegidas) |
| `ecosistemasDisponiblesHistorico` / `transportadorasDisponiblesHistorico` | Opciones de los filtros del histórico, sacadas de los lotes cargados |

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
| `HistorialDespachos` | GET con `pipeline` (`$match` por tienda, rango de fechas o `IdLote $in`; `$group` por `IdLote` con `$addToSet` de `Transportadora`) | `getResumenLotes(rango, idLotes?)` |
| `HistorialDespachos` | GET con `pipeline` (`$match` `Guia $in` + `$project` de `Guia`, `IdLote`) | `buscarGuiasEnHistorial()` |
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
  ├─► agrupa por NombreTienda → chunks de MAX_GUIAS_POR_PDF
  └─► por chunk: generarPdfDespacho() → store.exportarLote(chunk, tienda)
        ├─► POST a HistorialDespachos (IdLote, FechaGeneracion, DespachoGenerado:true)
        ├─► DELETE tolerante del borrador
        └─► si falla la limpieza → idsPendientesDeLimpiar + aviso de reintento

Pestaña "Histórico" → cambiarTab('historico') → store.cargarLotes()
  ├─► getResumenLotes(rango) [pipeline $group sobre HistorialDespachos]
  ├─► filtros de ecosistema / tienda / transportadora client-side
  ├─► buscarGuiasHistorico() [modo Lote o Guía, ver abajo]
  ├─► abrirDetalleLote(idLote) → getGuiasDeLote(idLote)
  │     └─► exportarSubsetDeLote() [genera PDF, no archiva de nuevo]
  ├─► regenerarPdf(idLote) [en la lista, sin abrir detalle: solo regenera]
  └─► marcar lotes → "Unir lotes" → aviso de confirmación → exportarLotesSeleccionados()
```

---

### Histórico de lotes

Pestaña con la lista de PDF ya generados (un lote por fila). En PC es una tabla; en celular, tarjetas.

**Columnas:** fecha y hora de generación, `IdLote`, tienda, transportadora, guías, recaudo y acciones (descargar / regenerar PDF).

- **Fecha y hora:** `formatearFechaHistorico()` (`23 jul 2026`) y `formatearHora()` (`3:45 p. m.`). Se formatean en **UTC** porque `fechaColombia()` ya guarda la hora de Colombia aunque el ISO termine en `Z`; convertirla otra vez restaría 5 horas más.
- **Transportadora:** etiqueta de color por `claseTransportadora()`, con los mismos colores del modal de creación de pedidos: Interrapidísimo azul, Envía rojo, Servientrega verde, TCC naranja, el resto gris.
- **Por página:** menú propio "Ver 10 / 25 / 50 / 100" (`OPCIONES_PAGE_SIZE_HISTORICO`). Existe uno para la lista de lotes y otro para el detalle del lote. En PC se abre hacia arriba (la barra está al pie de la tabla) y en celular como hoja inferior. Reemplaza al `<select>` nativo, cuya lista se salía de la pantalla en celular.

#### Filtros

Orden en pantalla: **Fecha, Ecosistema, Tienda, Transportadora**.

| Filtro | Dónde se aplica | Detalle |
|---|---|---|
| Fecha | Servidor (`getResumenLotes(rango)`) | Calendario propio. Un clic = un día; dos clics = rango (`RangoFechas`, inclusivo). El `$match` va de `desde T00:00:00.000Z` a `hasta T23:59:59.999Z` |
| Ecosistema | Cliente | Marca/desmarca todas las tiendas del ecosistema. La casilla queda a medias (`indeterminate`) si solo algunas están marcadas (`getEcoEstadoHistorico()` → `'all' \| 'some' \| 'none'`) |
| Tienda | Cliente | Multi-select con buscador. Solo aparece si hay más de una tienda en los lotes cargados |
| Transportadora | Cliente | Multi-select. El lote pasa si alguna de sus `Transportadoras` está elegida |

Las opciones de los menús usan la misma casilla propia de las tablas (`.despacho__check`: 18px, azul con ✓ al marcarla, rayita cuando está a medias) y la opción marcada se resalta en azul suave.

#### Buscador de guías (Lote | Guía)

Un solo campo con dos modos (`modoBusquedaGuias`). Cambiar de modo limpia la búsqueda anterior.

| Modo | Entrada | Resultado |
|---|---|---|
| **Lote** | Una o varias guías separadas por coma | La tabla muestra solo los lotes donde están esas guías. Arriba sale el aviso "Mostrando los lotes de las guías buscadas", con las **no encontradas** y el botón "Ver todos los lotes" |
| **Guía** | Una sola guía | Abre directo el detalle de su lote mostrando **solo esa guía** (`guiaDestacada`), con opción de ver todas. Si se escriben varias, avisa "Busca una sola guía" |

- **Pegar desde Excel:** `onPegarGuias()` detecta saltos de línea o tabulaciones en lo pegado y convierte la columna en una lista separada por `, ` sin duplicados.
- **Búsqueda en todo el historial**, no solo en los lotes cargados: `store.buscarGuiasEnHistorico()` → `repository.buscarGuiasEnHistorial()` busca `Guia $in` con las guías como número **y** como texto (por documentos antiguos) y respeta las tiendas asignadas. La guía puede volver como Int64 extendido (`{ $numberLong }`), por eso se normaliza con `guiaKey()`.
- Con los `IdLote` encontrados, `store.mostrarLotesDeBusqueda()` llama a `getResumenLotes(null, idLotes)` y guarda el resultado en `_lotesBusquedaGuias`, que reemplaza la lista mientras la búsqueda esté activa. Los filtros de tienda/transportadora siguen aplicando encima.

#### Unir lotes

1. Se marcan uno o varios lotes con la casilla (también "seleccionar los de esta página").
2. Debajo de los filtros aparece una **franja azul** con: "N lotes seleccionados", pastillas de Tiendas, Guías y Recaudo (`resumenLotesSeleccionados`), el enlace **Quitar selección** y el botón **Unir lotes**.
3. Al pulsar Unir lotes se abre un **aviso de confirmación propio** (no SweetAlert), con los mismos datos en un cuadro de tres columnas y los botones **Cancelar** / **Sí, unir lotes**. Se cierra con Esc, clic fuera o Cancelar. Su estilo sigue el aviso de salida del modal de direcciones, con acento azul.
4. `responderUnion(true)` → `exportarLotesSeleccionados()`: trae las guías de cada lote, las junta y genera el PDF en partes de `MAX_GUIAS_POR_PDF`. **No archiva de nuevo ni crea un `IdLote` nuevo.**

---

### Encabezado

Tarjeta blanca con:

- Ícono de camión, título y la etiqueta ámbar **"N guías pendientes"**.
- Botones Carga masiva, Limpiar y Exportar PDF, solo en el encabezado.
- Franja de **pasos ① Escanea o carga las guías → ② Revisa y filtra por tienda → ③ Exporta el PDF y entrégalo** (`#rd-pasos`). **Solo en PC**: en celular se oculta.

---

### Helpers

Todo vive en `helpers/`, aparte del componente y la carpeta `models/`.

**`despacho.store.ts` — métodos públicos de `DespachoStore` no listados arriba:**

| Método | Qué hace |
|---|---|
| `iniciar()` | Dispara `getInventario()`, `getPedidosPrefetch()` y `getMapaTiendas()`; solo el inventario bloquea el loading, el resto corre en paralelo sin bloquear el render |
| `procesarGuia(guiaCruda)` | Flujo del pistolero: `obtenerPedido()` → `validarGuia()` → inserción optimista → `insertarRegistros()` |
| `validarLote(guias)` | Flujo de la carga masiva: resuelve cache-first, batch de 300 (`buscarPedidosPorGuias`) para lo faltante, valida cada una con `validarGuia()` |
| `confirmarLote(registros)` | `insertarRegistros()` + `refrescarInventario()` de los válidos ya calculados por `validarLote()` |
| `eliminarDelBorrador(registro)` / `eliminarVariosDelBorrador(registros)` | `DELETE` individual o en chunks de 300 (`Guia=a,b,c` → `$in` en el backend) |
| `exportarLote(registros, tienda)` | Archiva en `HistorialDespachos` (`IdLote`, `FechaGeneracion`, `DespachoGenerado:true`) y limpia el borrador; guarda `idsPendientesDeLimpiar` si la limpieza falla parcialmente |
| `reintentarLimpieza()` | Reintenta el `DELETE` tolerante sobre `idsPendientesDeLimpiar()`, sin volver a archivar ni generar PDF |
| `cargarLotes()` / `setFechaHistorico(rango)` / `setPaginaHistorico()` / `setPageSizeHistorico(n)` | Estado del histórico: carga, filtro de fecha por día o rango (server-side), paginación y lotes por página (vuelve a la página 1) |
| `setTiendasHistorico()` / `setTransportadorasHistorico()` / `toggleEcosistemaHistorico(eco)` / `getEcoEstadoHistorico(eco)` | Filtros client-side del histórico |
| `buscarGuiasEnHistorico(guias)` | Devuelve `{ lotePorGuia, noEncontradas }` buscando en todo `HistorialDespachos` |
| `mostrarLotesDeBusqueda(idLotes)` / `limpiarBusquedaGuias()` | Activa o quita la lista de lotes de la búsqueda |
| `getGuiasDeLote(idLote)` | Delegado directo a `repository.getGuiasDeLote()`, usado por `regenerarPdf()`, por el detalle de lote y por Unir lotes |
| `obtenerPedido(guia)` (privado) | Cache-first: si `guiaKey(guia)` está en `_pedidosCache` y el prefetch ya terminó, la retorna; si no, cae a `repository.buscarPedidoPorGuia()` y la agrega al cache |

**`despacho.repository.ts` — capa HTTP, sin lógica de negocio:**

| Método | Colección/operación |
|---|---|
| `getPedidosPrefetch()` | GET `PedidosInter` proyectado (`campos=...`), filtrado por tienda asignada. Sin pipeline agregado |
| `buscarPedidoPorGuia(guia)` | GET `PedidosInter` por `Numeropreenvio` único |
| `buscarPedidosPorGuias(guias)` | GET `PedidosInter` con `Numeropreenvio=a,b,c` (`$in` en backend) |
| `getMapaTiendas()` | GET `Tienda` → agrupa por `Ecosistema` en memoria |
| `getInventario()` / `insertarRegistros()` / `eliminarRegistroPorGuia()` / `eliminarRegistrosTolerante()` | CRUD sobre `InventarioAdmision` |
| `archivarLote(guias)` | POST `HistorialDespachos` + `eliminarRegistrosTolerante()` de esas guías en el borrador |
| `getResumenLotes(rango, idLotes?)` | GET `HistorialDespachos` con `pipeline`: `$match` (tiendas, rango de `FechaGeneracion` y/o `IdLote $in`) → `$group` por `IdLote` (`$first` de `NombreTienda`/`FechaGeneracion`, `$sum` de guías/recaudo, `$addToSet` de `Transportadora`), stages en base64 |
| `buscarGuiasEnHistorial(guias)` | GET `HistorialDespachos` con `pipeline` `$match` `Guia $in` (número y texto) + `$project` `{ Guia, IdLote }` |
| `getGuiasDeLote(idLote)` | GET `HistorialDespachos` filtrado por `IdLote` |

**`despacho-pdf.helper.ts` — generación del PDF, sin llamadas HTTP:**

| Función | Qué hace |
|---|---|
| `generarPdfDespacho(registros)` | Punto de entrada. Orquesta carga de imágenes, barcodes, construcción de HTML, render a canvas y guardado del PDF |
| `cargarLogos(registros)` (privado) | Resuelve en paralelo el logo de cada transportadora presente (`LOGOS_TRANSPORTADORA`, match por substring en mayúsculas) |
| `generarBarcodes(guias)` (privado) | Un `CODE128` por guía vía `JsBarcode` sobre un canvas en memoria (nunca toca el DOM visible). Recibe la guía ya pasada por `guiaParaMostrar()`, así el código de Envía incluye el 0 y la pistola lo lee igual que la guía impresa |
| `construirHtml(...)` (privado) | Arma el documento agrupado por transportadora, con paginación cada `POR_PAGINA = 14` filas y firmas solo en la última página de cada grupo. Las cajas de logo de la cabecera miden 150px de ancho |
| `cargarImagenBase64(src)` (privado) | Convierte una imagen local a base64 vía canvas, para que `html2canvas` la renderice sin problemas de CORS. Agranda hasta 3x (con suavizado alto) las imágenes de menos de 1200px de ancho para que no se vean pixeladas en el PDF |
| `renderizarHtmlACanvas(html)` (privado) | Inyecta el HTML en un `<iframe>` oculto fuera de la ventana, espera fuentes (máx. 3 s) y captura con `html2canvas` |
| `guardarPDF(canvas, tienda)` (privado) | Corta el canvas en páginas A4 (JPEG 95%) y descarga `RelacionDespacho_<tiendaSlug>_<fecha>_<hora>.pdf` |

> El logo de Envía (`assets/images/logos/actuales/envia.png`) es un PNG de 2000×403 con fondo transparente y la frase en blanco, pensado para ir directo sobre el azul de la cabecera.

---

### Bloqueo de export mezclado

Si la fuente a exportar contiene guías de más de una tienda (`grupos.size > 1`) y el usuario no aplicó ningún filtro de tienda (`tiendasFiltro.length === 0`), `exportToPDF()` bloquea la operación con una alerta. Esto evita que un usuario con acceso a "Todas" las tiendas archive/exporte por error guías de una tienda que no es la que pretendía despachar. Si el usuario sí seleccionó tiendas en el multi-select, el export continúa generando un PDF y un `IdLote` separado por cada tienda.

---

### Escaneo por cámara (mobile)

El botón `rd-scan-btn` solo es visible en `@media (max-width: 768px)`. Abre `<app-escaner-camara>` ([EscanerCamara](../../../components/escaner-camara.md)), un componente genérico y reutilizable que emite `(scanSuccess)` con el valor leído; ese evento se conecta directo a `onGuiaEnter()`, el mismo flujo que el pistolero de texto/USB.

---

### Tour guiado

`app-tour-guiado` es un componente genérico (`src/app/components/tour-guiado/`) reutilizado en varios módulos. Este módulo define 11 pasos en `tourSteps` y recorre las dos pestañas: cada paso abre su pestaña con `onBeforeStep` y al salir del último (`onLeaveStep`) vuelve a Pendientes.

| # | Id | Pestaña | Qué explica |
|---|---|---|---|
| 1 | `rd-pasos` | Pendientes | Los tres pasos del despacho |
| 2 | `rd-pistolero` | Pendientes | Pistolero / digitar guía + Enter |
| 3 | `rd-scan-btn` | Pendientes | Escanear con la cámara |
| 4 | `rd-btn-masiva` | Pendientes | Carga masiva |
| 5 | `rd-filtros` | Pendientes | Filtros |
| 6 | `rd-tabla` | Pendientes | Guías pendientes (y el 0 de Envía) |
| 7 | `rd-btn-pdf` | Pendientes | Exportar PDF |
| 8 | `rd-tabs` | Histórico | La pestaña Histórico |
| 9 | `rd-h-filtros` | Histórico | Filtros del histórico (fecha por día o rango) |
| 10 | `rd-h-buscador` | Histórico | Buscador Lote / Guía |
| 11 | `rd-h-tabla` | Histórico | Lotes, por página, PDF y Unir lotes |

Para los pasos del histórico se usa `mostrarHistoricoParaTour()` (privado), que cierra el detalle de lote si estaba abierto y abre la pestaña. El componente del tour omite los pasos cuyo elemento no existe o está oculto, salvo los que tienen `onBeforeStep` (ese elemento aparece después de cambiar de pestaña). El botón del tour (`.despacho__btn-tour`) se oculta en celular, donde además los pasos ①②③ no se ven.

---

### Responsive / mobile

Todo está en `styles/relacion-despacho.mobile.scss`, en tres cortes:

**Tablet (≤860px):** encabezado y barras en columna.

**Celular (≤768px):**

- La vista ocupa todo el ancho de la pantalla con `margin-inline: calc(50% - 50vw)` (sin márgenes negativos fijos), 14px a cada lado, sin relleno arriba y `overflow-x: clip` para que la página nunca se arrastre de lado. Separación de 18px entre bloques.
- **Sin cajas dentro de cajas:** barras y recuadros de tabla sin borde ni sombra; filtros y tarjetas van directo sobre el fondo.
- **Tablas → tarjetas** (`.despacho__cards` / `.despacho__item`) en las tres tablas: pendientes, lotes del histórico y detalle del lote. La casilla de selección va en la tarjeta.
- Encabezado: botones de 44px de alto, pasos ①②③ ocultos, botón del tour oculto.
- Filtros en **dos columnas** (42px de alto, texto a la izquierda y flecha o X a la derecha); el buscador de guías del histórico abajo, a todo el ancho.
- Pistolero y buscador con `flex: none` y 44px de alto (en la barra en columna `flex: 1` los encogía).
- Los menús (fecha, ecosistema, tienda, transportadora, "Ver N") pasan de panel flotante a **hoja inferior fija con fondo oscuro**. `z-index` de `20100` / `20050` para quedar por encima del sidebar del layout (`z-index: 20000`).
- Franja de Unir lotes en columna: texto, datos y botones a todo el ancho. El aviso de confirmación también tiene botones a todo el ancho.
- Barra inferior de la tabla en dos filas (información y "Ver N" arriba, paginador abajo); lo que solo sirve en PC lleva `.despacho__solo-pc`.

**≤400px:** ajustes de tamaño para celulares pequeños.

---

### Pruebas

58 pruebas (Karma + Jasmine):

| Archivo | Pruebas | Qué cubre |
|---|---|---|
| `helpers/despacho.rules.spec.ts` | 20 | Validaciones, `guiaKey`, `guiaParaMostrar` (0 de Envía), ids de lote |
| `helpers/despacho.store.spec.ts` | 14 | Filtros del histórico (tienda, transportadora, ecosistema a medias), por página, rango de fechas y búsqueda de guías (incluida la guía como Int64) |
| `helpers/despacho.repository.spec.ts` | 6 | Pipelines que se envían al backend: tiendas del usuario, rango de fechas, `IdLote $in`, guía como número y texto |
| `relacion-despacho.component.spec.ts` | 17 | Calendario de rango, un solo menú abierto, pegado desde Excel, modos Lote/Guía, color de transportadora, hora sin restar 5 horas, ids del tour |
| `components/carga-masiva/carga-masiva.component.spec.ts` | 1 | Creación del componente |

`helpers/despacho.repository.fake.ts` es un repositorio falso **solo para las pruebas**: evita llamadas HTTP reales en los `should create`.

```bash
npx ng test --watch=false --browsers=ChromeHeadless --include="src/app/views/logistica/relacion-despacho/**/*.spec.ts"
```

---

### Historial de cambios

| Fecha | Cambio |
|---|---|
| 2026-07-03 | Creación del módulo original: pistolero, tabla, PDF agrupado por transportadora (arquitectura reemplazada) |
| 2026-07-27 | Reescritura sobre arquitectura Store/Repository/Rules, carga masiva vía Excel, cache-first de `PedidosInter`, escáner por cámara, pestaña de histórico agrupado por lote con detalle y regeneración de PDF |
| 2026-10-26 | Histórico con fecha y hora, rango de fechas, columna y filtro de transportadora, filtro de ecosistema, "Ver 10/25/50/100", buscador Lote/Guía en todo el historial, Unir lotes con confirmación. 0 inicial de Envía en tablas, PDF y código de barras; logo de Envía nuevo y logos del PDF en alta calidad. Encabezado nuevo con pasos, tour de 11 pasos por las dos pestañas, versión completa para celular, estilos en `styles/` y 58 pruebas |

---

### Observaciones

- El PDF se genera con `jsPDF` + `html2canvas` + `JsBarcode`, renderizando el HTML dentro de un `<iframe>` oculto fuera de la ventana visible para aislarlo de estilos globales; se trocea el canvas en páginas A4.
- El PDF se genera **antes** de archivar en `HistorialDespachos`: si `generarPdfDespacho()` falla, ese chunk nunca se mueve del borrador.
- Si `exportarLote()` archiva correctamente pero el `DELETE` del borrador falla parcialmente, las guías quedan en `idsPendientesDeLimpiar()` y el usuario puede reintentar la limpieza sin volver a generar el PDF ni duplicar el archivado.
- El 0 de Envía no se guarda en la base de datos; si alguna vez se cambia cómo se guarda la guía, revisar `guiaParaMostrar()` para no poner el 0 dos veces (hoy ya lo evita: solo lo agrega si falta).
- El filtro de tiendas asignadas del buscador de guías se aplica en el frontend (en el `$match` que arma el repositorio); el backend genérico no lo valida por su cuenta.
