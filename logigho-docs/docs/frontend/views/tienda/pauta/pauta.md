---
autor: Adalberto González
fecha_creacion: 2026-04-24
ultima_actualizacion: 2026-10-14
estado: Desarrollo
nivel: 2
---

# Vista: Pauta

**Autor:** Adalberto González  

**Selector:** `app-pauta`  

**Ubicación:** `SitioLogiGho/src/app/views/tienda/pauta`

---

## ¿Qué hace?

Vista de gestión de inversión publicitaria por tienda. Permite listar, buscar, filtrar por fechas, editar inline y descargar los registros de pauta (Meta, TikTok, Google y Otros) filtrados por las tiendas asignadas al usuario. También permite importar nuevos registros desde un archivo Excel con previsualización editable (ver [Importacion Pauta](../../../components/importacion-pauta.md)) y descargar la plantilla de importación desde S3.

Carga **todos** los registros de las tiendas del usuario (el backend los entrega por páginas), así el contador, los totales, el filtro y la descarga trabajan siempre sobre el conjunto completo. Los montos solo aceptan números enteros, y los valores viejos mal guardados (ej. `1.500.000`) se ven en rojo sin romper los totales.

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
├── pauta.component.ts                    → Tabla, filtros, edición, totales, descarga y tour guiado
├── pauta.component.html
├── pauta.component.scss                  → Estilos de la vista; incluye los dos mixins de estilos/
├── pauta.component.spec.ts               → 9 pruebas (carga por páginas, reintento, estadoMonto, totales)
├── models/
│   └── pauta.models.ts                   → Interfaces y columnas del módulo (ver Modelos)
├── repository/
│   └── pauta.repository.ts               → Único que llama al backend (ver Repositorio)
├── services/
│   ├── lectura-excel-pauta.service.ts    → Lee el Excel y revisa encabezados (lo usa el modal)
│   ├── validacion-filas-pauta.service.ts → Valida cada fila y verifica tiendas (lo usa el modal)
│   └── aviso-toast.service.ts            → Aviso pequeño arriba a la derecha
├── directives/
│   └── campo-solo-numeros.directive.ts   → `appCampoSoloNumeros`: input que solo acepta dígitos
└── estilos/
    ├── _aviso-carga-registros.scss       → Barra "Cargando registros… X de N" y aviso de carga incompleta
    └── _diseno-celular.scss              → Tabla como tarjetas, calendario como hoja y paginación compacta
```

> El modal de importación sigue en su lugar original (`src/app/components/importacion-pauta`), pero usa el modelo, el repositorio, los servicios y la directiva de esta carpeta.

---

## Secciones de la vista

| # | Sección | Descripción |
|---|---|---|
| 1 | **Cabecera** | Título "Pautas", subtítulo, botones `Descargar Plantilla`, `Descargar Registros`, `Importar` y botón `?` del tour guiado. En celular los botones dicen solo `Plantilla` y `Registros` |
| 2 | **Modal de importación** | `app-importacion-pauta` — se abre con el botón Importar |
| 3 | **Aviso de carga** | Barra "Cargando registros… X de N" mientras llegan las páginas. Si alguna no llega: aviso rojo "Se cargaron X de Y registros" con botón `Reintentar` |
| 4 | **Tarjetas de resumen** | 4 stat-cards: Total Meta, Total TikTok, Total Google y Otros (suman el conjunto filtrado activo; se atenúan mientras carga) |
| 5 | **Barra de herramientas** | Buscador con debounce de 220 ms, filtro de fechas `Desde` / `Hasta` con calendario propio y `Limpiar`, y contador de registros filtrados |
| 6 | **Tabla de pautas** | Listado paginado con edición inline por fila: Fecha, ID Tienda, Tienda, Meta, TikTok, Google, Otros y Acción |
| 7 | **Pie de tabla / Paginación** | Selector de registros por página (5, 10, 25, 50) y controles de paginación con elipsis. En celular: "‹ Página X de N ›" |
| 8 | **Tour guiado** | `app-tour-guiado` con 10 pasos sobre la vista |

---

## Propiedades del componente

### Datos

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `pauta` | `any[]` | `[]` | Lista activa (puede estar filtrada). La tabla y las stat-cards operan sobre esta lista. |
| `pautaOriginal` | `any[]` | `[]` | Todos los registros cargados desde BD. Fuente de verdad para el filtro. |
| `modoEdicion` | `Record<string, boolean>` | `{}` | Indexado por `_id`. `true` si la fila está en modo edición. |
| `registroBackup` | `any` | `{}` | Copia del registro antes de editar, indexada por `_id`, para poder cancelar. |

### Carga de registros

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `PAGINAS_EN_PARALELO` | `number` | `8` | Páginas que se piden al backend al mismo tiempo. |
| `cargando` | `boolean` | `false` | `true` mientras llegan las páginas. |
| `progresoCarga` | `{ actual, total }` | `{0, 0}` | Páginas recibidas de las que hay. Alimenta la barra de avance. |
| `cargaIncompleta` | `{ cargados, total } \| null` | `null` | Si alguna página no llegó después del reintento: cuántos registros se cargaron de los que hay. |
| `cargaActual` | `number` | `0` | Número de la carga en curso. Si empieza otra (ej. después de importar), las respuestas de la anterior se ignoran. |

### Búsqueda y filtro

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `filtro` | `string` | `''` | Texto del buscador. Se evalúa contra todos los campos del registro. |
| `fechaInicio` / `fechaFin` | `string` | `''` | Rango de fechas en formato `YYYY-MM-DD`. |
| `activeCalendar` | `'inicio' \| 'fin' \| null` | `null` | Calendario abierto. Un clic fuera (`document:click`) lo cierra. |
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

### Modal y tour

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `isImpPautaModalOpen` | `boolean` | `false` | Controla la visibilidad del modal de importación. |
| `tourAbierto` | `boolean` | `false` | Controla el tour guiado. |
| `tourSteps` | `TourStep[]` | 10 pasos | Totales, buscador, fechas, tabla, editar, paginación, los tres botones de la cabecera y el botón del tour. |

---

## Interfaz de datos

El registro guardado está definido en [Modelos](models/pauta-models.md) como `RegistroPauta`. En la tabla se le suma el `_id`:

```typescript
interface RegistroPauta {
  Fecha: string;      // Fecha del registro (ej. "2026-04-01")
  Idtienda: number;   // ID numérico de la tienda
  Tienda: string;     // Nombre de la tienda
  Meta: number;       // Inversión en Meta Ads
  TikTok: number;
  Google: number;
  Otros: number;      // En la BD el campo se llama "Otros", no "Otro"
}
```

---

## Flujo de inicialización

```
ngOnInit()
  → fetchTableData()
      → Lee tiendas_asignadas de sessionStorage
      → PautaRepository.consultarPagina(1, tiendas)
          → lee NumeroPaginas y TotalRegistros
      → Pide las páginas 2..N de a 8 en paralelo (Promise.allSettled)
          → actualiza progresoCarga en cada página
          → las que fallen se reintentan una vez
      → Junta las páginas en orden → pautaOriginal
      → applyFilter() (conserva búsqueda y fechas del usuario)
      → Si faltó alguna página → cargaIncompleta
```

---

## Métodos

### `fetchTableData(): Promise<any[]>`
Carga todos los registros de pauta de las tiendas del usuario.

**Proceso:**
1. Lee `tiendas_asignadas` de `sessionStorage`.
2. Pide la página 1 con `PautaRepository.consultarPagina()` y lee cuántas páginas y registros hay.
3. Pide las demás de a `PAGINAS_EN_PARALELO`; las que fallen se reintentan una vez.
4. Junta todo en el orden del backend (más nuevos primero) y mapea los campos `_id`, `Fecha`, `Idtienda`, `Tienda`, `Meta`, `TikTok`, `Google` y `Otros`.
5. Vuelve a aplicar el filtro activo y recalcula totales y paginación.
6. Si alguna página no llegó, deja `cargaIncompleta` con cuántos registros se alcanzaron a cargar.

**Manejo de error:** si falla la primera página, `Swal.fire` de error "Error al cargar la pauta". Si mientras tanto empezó otra carga, la respuesta vieja se descarta (`cargaActual`).

---

### `filtrarTabla()`
Aplica un debounce de 220 ms antes de llamar a `applyFilter()`. Cancela el timeout anterior en cada keystroke para no ejecutar filtros intermedios.

---

### `applyFilter()` *(privado)*
Filtra `pautaOriginal` por el texto de `filtro` (contra todos los campos, mediante `Object.values`) y por el rango `fechaInicio` – `fechaFin`. Resetea la página a 1, invalida el caché y recalcula la paginación.

---

### `editarFila(id)`
Habilita el modo edición para la fila con ese `_id`.

**Proceso:**
1. Crea una copia del registro en `registroBackup[id]`.
2. Pone `modoEdicion[id]` en `true`.

---

### `guardarFila(id): Promise<void>`
Persiste los cambios de la fila en edición en la BD.

**Proceso:**
1. Valida que `Fecha`, `Tienda` e `Idtienda` no estén vacíos.
2. Revisa cada monto con `estadoMonto()` **antes** de convertir y sin tocar el registro: vacío → `0`, válido → `number`, inválido → error.
3. Si hay montos o `Idtienda` inválidos, avisa con `Swal.fire` qué campos tienen que ir solo con números y no guarda.
4. Llama a `PautaRepository.actualizar(id, registro)`.
5. Solo si el guardado fue exitoso actualiza la fila en pantalla, pone `modoEdicion[id]` en `false` e invalida el caché de totales.

**Manejo de error:** `Swal.fire` de error si la llamada falla; la fila queda como estaba.

---

### `cancelarEdicion(id)`
Descarta los cambios de la fila restaurando el registro desde `registroBackup[id]` y deshabilita el modo edición.

---

### `estadoMonto(value): 'vacio' | 'valido' | 'invalido'`
Clasifica un monto para mostrarlo, sumarlo o validarlo.

| Estado | Cuándo | Cómo se ve en la tabla |
|---|---|---|
| `vacio` | `null`, `undefined` o texto vacío | `—` |
| `valido` | Número entero ≥ 0, o texto con solo dígitos | Con formato de moneda |
| `invalido` | Puntos, comas, espacios, letras, decimales o negativos | El valor tal cual, en rojo, con un `title` que explica cómo corregirlo |

---

### `calcularTotal(campo): number`
Suma los valores **válidos** del `campo` en `this.pauta` (datos filtrados activos). Un monto mal guardado no vuelve `NaN` el total.

**Optimización:** Usa `totalsCache` indexado por `campo_version`. Solo recalcula cuando `totalsCacheVersion` cambia (al filtrar o guardar). Evita recalcular en cada ciclo de detección de cambios de Angular.

---

### `totalRegistro(item): number`
Suma la inversión de un registro (solo montos válidos). Se muestra en la tarjeta de cada registro en celular.

---

### `onPlantillaClick()`
Descarga el archivo `Pauta.xlsx` desde el bucket S3 `logigho-plantillas` vía `PautaRepository.obtenerPlantilla()`. Convierte la respuesta base64 a `Blob` y lo descarga mediante un enlace temporal.

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

### Calendario de fechas

| Método | Descripción |
|---|---|
| `openCalendar(which)` | Abre el calendario de `Desde` o `Hasta` en el mes de la fecha elegida (o el actual). Si ya estaba abierto, lo cierra. |
| `closeCalendar()` | `@HostListener('document:click')`: cierra el calendario al hacer clic fuera. |
| `prevMonth()` / `nextMonth()` | Navegan entre meses. |
| `selectDay(day)` | Guarda la fecha en `YYYY-MM-DD`, cierra el calendario y aplica el filtro. |
| `limpiarFechas()` | Quita el rango de fechas. |

---

### `openImportInveModal()` / `closeImpInveModal()`
Controlan `isImpPautaModalOpen` para abrir y cerrar el modal `app-importacion-pauta`.

---

## Subcomponentes

| Componente | Selector | Cuándo se monta | Evento de retorno |
|---|---|---|---|
| `ImportacionPautaComponent` | `app-importacion-pauta` | Siempre (controlado por `[isOpen]`) | `(closeEvent)` → `closeImpInveModal()` / `(refreshDataEvent)` → `fetchTableData()` |
| `TourGuiadoComponent` | `app-tour-guiado` | Siempre (controlado por `[isOpen]="tourAbierto"`) | `(closed)` → `tourAbierto = false` |

**Directiva:** los inputs de montos en edición usan [`appCampoSoloNumeros`](directives/campo-solo-numeros-directive.md).

---

## Servicios utilizados

| Servicio | Métodos usados | Propósito |
|---|---|---|
| [`PautaRepository`](repository/pauta-repository.md) | `consultarPagina()`, `actualizar()`, `obtenerPlantilla()` | Carga por páginas, actualización de registros y descarga de la plantilla |
| `Router` | — | Inyectado pero sin uso activo |
| `ActivatedRoute` | — | Inyectado pero sin uso activo |

---

## Endpoints que consume

Todos pasan por [`PautaRepository`](repository/pauta-repository.md).

| Método | Ruta | Cuándo |
|---|---|---|
| `GET` | `metodoGenerico?coleccion=Pauta&Tienda={tiendas}&mcomp=2&campos=Fecha,Idtienda,Tienda,Meta,TikTok,Google,Otros` (página N) | Al inicializar, tras importar y al reintentar |
| `PUT/PATCH` | `metodoGenerico?coleccion=Pauta` (vía `actualizarGenerico`) | Al guardar una fila editada |
| `GET` | S3 bucket `logigho-plantillas`, key `Pauta.xlsx` | Al descargar la plantilla |

---

## Estados de la vista

| Estado | Descripción | Qué muestra |
|---|---|---|
| Cargando | `fetchTableData()` en curso | Barra "Cargando registros… X de N", totales atenuados y "Cargando registros…" en la tabla vacía |
| Carga incompleta | Alguna página no llegó tras el reintento | Aviso rojo "Se cargaron X de Y registros. Los totales no están completos." con `Reintentar` |
| Con datos | Registros cargados | Tabla con paginación y stat-cards |
| Filtrado activo | Texto en el buscador o rango de fechas | Tabla reducida; stat-cards reflejan solo los filtrados |
| Fila en edición | `modoEdicion[id] = true` | Inputs inline reemplazan los spans; botones Guardar/Cancelar. En celular la tarjeta queda con borde azul |
| Monto inválido | Dato viejo mal guardado (ej. `1.500.000`) | El valor en rojo; no suma en los totales |
| Sin resultados | Filtro no coincide | Fila `"No se encontraron registros de pauta."` |
| Error de carga | Falla la primera página | `Swal.fire` de error |

---

## Diseño en celular

En pantallas de menos de 768 px (`estilos/_diseno-celular.scss`):

- `Importar` ocupa todo el ancho y debajo quedan `Plantilla` y `Registros`.
- La tabla pasa a **tarjetas** solo con CSS: cada `td` tiene una clase (`c-fecha`, `c-id`, `c-monto`, `c-total`…) y un `data-etiqueta`; los montos salen en cajitas con la barra del color de la plataforma y abajo el **total del registro**.
- El calendario sale como hoja desde abajo, con un fondo (`.cal-fondo`) que lo cierra al tocar fuera.
- La paginación muestra "‹ Página X de N ›".
- En 400 px o menos las tarjetas de resumen se apilan.
- El contenedor ocupa todo el ancho (`.pauta-container` con `width: 100vw`), porque el layout deja unos 40 px por lado en todas las páginas.

---

## Changelog de la vista

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-04-24 | Adalberto González | Personalizacion de estilos en el scss de el componente |
| 2026-04-24 | Adalberto González | Peronalizacion de la tabla que muestra los datos, ahora podemos filtrar y paginar |
| 2026-05-16 | Adalberto González | Se volvio a añadir el editar fecha en la tabla de el modulo |
| 2026-10-14 | Yadilton Torres | Corrección del campo `Otros` (se leía como `Otro`) en carga, tarjeta, columna, edición y descarga |
| 2026-10-14 | Yadilton Torres | Montos solo con números enteros: `estadoMonto()`, totales que solo suman válidos y edición con `appCampoSoloNumeros` |
| 2026-10-14 | Yadilton Torres | Carga de todas las páginas (8 en paralelo, `mcomp: 2` + `campos`) con barra de avance y aviso de carga incompleta |
| 2026-10-14 | Yadilton Torres | Al guardar se valida antes de convertir (sin dejar `NaN` en la fila) y la fila solo cambia si el guardado fue exitoso |
| 2026-10-14 | Yadilton Torres | Llamadas al backend movidas a `PautaRepository`; estilos nuevos en `estilos/` |
| 2026-10-14 | Yadilton Torres | Diseño en celular (tarjetas, calendario como hoja, paginación compacta) y tour guiado |

---

## Observaciones

- **`Router` y `ActivatedRoute` sin uso:** Inyectados en el constructor pero sin llamadas activas. Pueden eliminarse.
- **`productosFiltrados`, `isLoading`, `isImage`, `isPdf`, `fileToUpload`, `isHovered`, `fileBlob`, `fileUrl`, `extensionArchivo`, `fileName`:** Propiedades declaradas pero sin uso activo en la vista. Son residuo de un template anterior y pueden eliminarse para reducir ruido.
- **Sin confirmación antes de guardar:** `guardarFila()` persiste directamente sin `Swal.fire` de confirmación previo, a diferencia del flujo de eliminación en `PortafolioComponent`.
- **Filtro de tiendas solo en frontend:** la consulta se filtra por el **nombre** de las tiendas de `sessionStorage.tiendas_asignadas`. La seguridad real (que un usuario no vea pauta de otra tienda) debe validarse en el backend.
- **Sin control por rol:** cualquier usuario con el módulo en su menú puede importar, editar y descargar.
- **Los estilos nuevos van en mixins** (`@mixin estilos`) que `pauta.component.scss` incluye con `@include` en el mismo lugar donde estaban. Con `@use` arriba del archivo cambiaría el orden de la cascada y el diseño de celular perdería contra el de computador.
