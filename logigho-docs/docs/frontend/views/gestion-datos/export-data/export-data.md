---

## Autor: Adalberto González

Fecha creacion: 2026-05-13
Estado: produccion
Tipo: vista

# Vista: ExportDataComponent

**Selector:** `app-export-data`  
**Ubicación:** `src/app/views/export-data/export-data.component`  
**Acceso:** Autenticado | Guard: `AuthGuard`

---

## ¿Qué hace?

Pantalla principal del módulo de exportación ("Exportación Dinámica de Datos"). Permite al usuario seleccionar una colección de MongoDB, elegir el formato de salida (Excel, CSV, JSON o PDF), las columnas a incluir, un rango de fechas y filtros avanzados por campo (incluida la búsqueda por varias guías). Antes de exportar muestra un resumen de cómo queda la exportación y cuántos registros trae. Al confirmar, abre una nueva pestaña con `ExportRunnerComponent` pasando toda la configuración via URL y `localStorage`.

En las colecciones grandes (Pedidos y Liquidaciones) limita el rango de fechas y los registros según el formato, para que el navegador alcance a armar el archivo.

---

## Ruta

| Ruta | Guard | Parámetros de URL |
| --- | --- | --- |
| `/app/export-data` | `AuthGuard` | — |

---

## Partes de la pantalla

| Parte | Qué muestra |
| --- | --- |
| **Encabezado** | Título "Exportación Dinámica de Datos" y la fila **"Ir directo a"** con accesos a Columnas, Fechas, Filtros, Exportar e Información de uso. A la derecha el botón **"¿Cómo funciona? Ver recorrido"** (tour), que late la primera vez que el usuario entra. |
| **Sin colección** | Solo el aviso **"Primero selecciona una colección"**. No se muestran las demás tarjetas y en "Ir directo a" quedan solo Información de uso y el tour. |
| **Seleccionar colección** | Lista con buscador. Solo salen las colecciones que el rol del usuario puede ver (campo `Roles` de `ListaColecciones`). |
| **Formato** | Lista propia con nivel de capacidad: **ALTO** (CSV), **MEDIO** (Excel, JSON), **BAJO** (PDF) y "Hasta N registros" en las colecciones con límite. En las demás dice **SIN LÍMITE**. Se maneja también con el teclado. |
| **Columnas** | "Todas" o "Seleccionar columnas específicas", con **buscador de columnas** (no distingue mayúsculas, tildes, espacios ni guiones bajos, marca lo que coincide y muestra un contador). |
| **Filtros por fecha** | Rangos Última semana, **Este mes**, **Último mes**, Últimos 3 meses, Últimos 6 meses y Todo, con la cantidad de registros debajo de cada botón (colecciones con límite). Fechas personalizadas con días tachados fuera del límite. Aviso del límite con el mensaje al equipo de desarrollo. |
| **Filtros avanzados** | Campo + valor + **Agregar**. El campo se elige también con el teclado. En Pedidos y Liquidaciones aparece primero el filtro **Ecosistema** (se pueden marcar varios). Los campos numéricos enteros aceptan **varios valores** (ej. varias guías) y las guías y preenvíos se revisan contra la base al agregarlos. Botones Ver Ejemplos (azul suave), Limpiar Todo (rojo suave) y Agregar (azul, como Exportar). |
| **Así queda tu exportación** | Resumen fijo encima de "Información de Uso": a la izquierda los pasos (Colección, Formato, Fechas, Columnas, Filtros) con "Cambiar"; a la derecha el panel **Registros a exportar** con número, barra de capacidad, fichas (Columnas, Celdas, Aún caben), estado y botón **Exportar**. |
| **Pastilla "↓ Ir a exportar"** | Aparece cuando el usuario cambia algo y los botones de exportar no están en pantalla. Lo baja hasta ellos y los hace brillar. |
| **Información de Uso** | Ayuda de la pantalla. En celular empieza plegada. |
| **Ayudas al pasar el mouse** | Tarjetas propias con la directiva [`appAyuda`](directives/ayuda-flotante-directive.md) en lugar del `title` del navegador: rangos (con sus fechas exactas), columnas, varias guías, quitar filtro, valores de filtros y botón Exportar. |

---

## Propiedades clave

| Propiedad | Tipo | Descripción |
| --- | --- | --- |
| `colecciones` | `Coleccion[]` | Lista de colecciones disponibles, cargada desde `ListaColecciones` al iniciar y filtrada por el campo `Roles` del usuario |
| `campoConfigs` | `CampoConfig[]` | Configuración de campos de la colección activa, cargada desde `ConfiguracionCamposColeccion`. Solo quedan los campos `ACTIVO` que el rol del usuario puede ver. Determina tipo de control, opciones, orden y visibilidad de cada campo en los filtros |
| `filtrosAvanzados` | `FiltroAvanzado[]` | Filtros que el usuario ha agregado. Cada entrada tiene `campo` y `valor` (con varios valores, separados por coma) |
| `camposMetadata` | `Map<string, CampoMetadata>` | Metadata por campo: si es fecha y su formato (`YYYY-MM-DD`, `DD/MM/YYYY`, `YYYY-MM-DD HH:mm:ss`). Se infiere de `campoConfigs` o de los datos reales de la colección |
| `campoSelectOptions` | `Map<string, {value,label}[]>` | Caché lazy de opciones para campos SELECT/MULTISELECT. Se puebla la primera vez que el usuario abre el dropdown de ese campo |
| `formatos` | `{ valor, nombre, icono, nivel, uso }[]` | Formatos de la lista con su nivel de capacidad (`ALTO`, `MEDIO`, `BAJO`) |
| `filtroFechaPredefinido` | `string` | Período predefinido seleccionado: `ultima_semana`, `este_mes`, `ultimo_mes`, `ultimos_3_meses`, `ultimos_6_meses`, `todo` |
| `puedeVerFiltroTodo` | `boolean` | Si es `false` (según rol), la opción `todo` no aparece en la UI y se fuerza un rango de fechas |
| `conteosRango` | `{ [rango]: number \| null }` | Registros de cada rango (Este mes, Último mes, 3 meses, 6 meses, Todo) con las tiendas del usuario. Solo en colecciones con límite |
| `conteoSeleccion` | `number \| null` | Registros de la selección exacta (fechas + filtros), contados en el backend cuando el conteo por rango no alcanza |
| `maxValoresPorFiltro` | `number` | Máximo de valores en un filtro de varios valores (ej. guías): **400** |
| `maxLargoFiltrosUrl` | `number` | Caracteres máximos de los filtros en la URL de `export-runner` cuando hay Ecosistema: **7000** (o el largo de las tiendas propias del usuario, si es mayor) |
| `revisandoGuias` | `boolean` | Se está revisando en el backend que las guías o preenvíos del filtro existan (el botón Agregar dice "Revisando…") |
| `maxColumnasPdf` | `number` | Máximo de columnas del PDF en las colecciones con límite: **10** |
| `mensajeEquipoDesarrollo` | `string` | Texto del mensaje al equipo de desarrollo (máximo 500 caracteres) |
| `esCelular` | `boolean` | Pantalla de hasta 576px (`matchMedia`). Cambia las listas a paneles desde abajo y usa el calendario del teléfono |
| `totalRegistros` | `number` | Total de registros de la colección con los filtros actuales. Se guarda en `localStorage` para que `export-runner` calcule cuántas páginas debe pedir |

---

## Servicios y endpoints

| Servicio | Método | Endpoint | Cuándo |
| --- | --- | --- | --- |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET /metodoGenerico?coleccion=ListaColecciones&Estado=ACTIVO` | `ngOnInit` — carga el listado de colecciones |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET /metodoGenerico?coleccion=ConfiguracionCamposColeccion&NombreColeccion=X&mcomp=2` | Al seleccionar colección — carga la configuración de campos (activos e inactivos; los inactivos se ocultan) |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET /metodoGenerico?coleccion=X&mcomp=2&lote=100&[filtros]` | Al seleccionar colección — obtiene `TotalRegistros` y una muestra para detectar campos sin configurar |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET /metodoGenerico?coleccion=X&mcomp=2&lote=1&campos=_id&[filtros]&fechasFiltro=...` | Conteos: 5 consultas a la vez al elegir colección (una por rango) y, al cambiar fechas o filtros, la de la selección exacta (espera 500 ms) |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET /metodoGenerico?coleccion=X&mcomp=2&lote=2000&campos=<campo>&[filtros]` | Lazy, al abrir un SELECT de tipo `LISTA` — trae los valores reales del campo para ajustar las opciones |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET /metodoGenerico?coleccion=Tienda&mcomp=2&lote=500&Estado=ACTIVO` | Lazy, al abrir un SELECT/MULTISELECT cuyo `CampoEtiquetaReferencia === 'NombreTienda'` |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET /metodoGenerico?coleccion=Tienda&mcomp=2&lote=8000&campos=NombreTienda,Ecosistema` | Lazy, la primera vez que se usa el filtro Ecosistema (se guarda para no repetirla) |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET /metodoGenerico?coleccion=X&mcomp=2&lote=8000&campos=<campo>&[FiltroReferencia]&<campo>=g1,g2,...` | Al agregar guías o preenvíos — revisa cuáles existen en las tiendas del usuario (sin fechas) |
| [`NotificacionesService`](../../../core/notificaciones-service.md) | `enviar()` | Inserta en `NotificacionesPlataforma` una notificación tipo `SOLICITUD_EXPORTACION` | Al enviar el mensaje al equipo de desarrollo |
| `AvisoNotificacionService` | `mostrar()` | — | Avisos del mensaje al equipo (enviado, espera, error, envío en preparación) |
| `DecompressionService` | `decompressZstd()` / `decompressGzip()` | — | Para descomprimir todas las respuestas del backend |

---

## Flujo principal

```
ngOnInit()
  -> esCelular = matchMedia('(max-width: 576px)')
  -> mostrarLatidoTour si el usuario nunca abrió el tour (localStorage 'export_tour_visto')
  -> actualizarPermisosRol()
       // determina puedeVerFiltroTodo según rol en sessionStorage
  -> cargarColecciones()
       // GET ListaColecciones → colecciones[] solo con las que permite el campo Roles
  -> [usuario selecciona colección]
  -> selectColeccion()
       -> cargarConfiguracionCampos()
            // GET ConfiguracionCamposColeccion → campoConfigs[] (ACTIVO + rol) ordenados por Orden ASC
            // los demás quedan en camposInactivos y no se muestran ni se exportan
       -> cargarColumnasDisponibles()
            // GET colección con lote=100 + FiltroReferencia → TotalRegistros
            // campos que existen en los datos pero no están configurados van al final
       -> agregarCampoEcosistema()       // solo PedidosInter y LiquidacionesLogigho
       -> ajustarAlLimiteExportacion()   // rango por defecto "Este mes"; PDF → columnas específicas
       -> contarRangos()                 // 5 conteos a la vez (solo colecciones con límite)

  -> ngDoCheck()
       -> revisarConteoSeleccion()       // si cambiaron fechas o filtros, cuenta de nuevo (500 ms)
       -> firma de la configuración      // si cambió y los botones no se ven → pastilla "Ir a exportar"

  -> [usuario abre dropdown de campo SELECT/MULTISELECT]
  -> ensureSelectOptions(campo)
       // Ecosistema (DINAMICO) → ecosistemas de la colección Tienda
       // LISTA → opciones configuradas, luego ajustadas con los valores reales del campo
       // COLECCION → GET coleccionReferencia (omite #VALOR)
       //   si CampoEtiquetaReferencia === 'NombreTienda':
       //     → usa NombreTienda como value (salvo campos de Id, ej. IdTienda)
       //     → filtra por tiendas_asignadas solo en el campo de la tienda dueña del registro

  -> [usuario agrega filtro]
  -> agregarFiltro()                         // async, devuelve true si lo agregó
       // valida que no existan duplicados de campo
       // campos de varios valores → limpiarVariosValores() (solo dígitos, sin repetidos, máx 400)
       // guía o preenvío → revisarIdentificadoresExistentes() (pregunta si faltan)
       // empuja a filtrosAvanzados[]

  -> exportarDatos()
       -> revisarFiltroSinAgregar()      // pregunta si quedó un filtro elegido sin "Agregar"
       -> validarLimiteExportacion()     // rango, registros y columnas del PDF (colecciones con límite)
       -> pedirOrientacionPdf()          // solo PDF
       -> resolverFiltrosParaEnvio()     // Ecosistemas → Tienda=a,b,c (revisa que quepan en la URL)
       // construye URLSearchParams con filtros base + filtros a enviar
       // guarda en localStorage:
       //   export_columns_<exportId>        → columnas seleccionadas
       //   export_total_registros_<exportId> → totalRegistros
       // limpia entradas de localStorage con más de 30 minutos de antigüedad
       // abre window.open('/app/export-runner?coleccion=X&formato=Y&exportId=Z&fechasFiltro=...&[filtros]')
```

---

## Rangos de fecha

| Rango | Desde | Hasta |
| --- | --- | --- |
| Última semana | Hoy menos 7 días | Hoy |
| **Este mes** | Día 1 del mes actual | Hoy |
| **Último mes** | Día 1 del mes anterior | Último día del mes anterior (en enero es diciembre del año anterior) |
| Últimos 3 meses | Hoy hace 3 meses | Hoy |
| Últimos 6 meses | Hoy hace 6 meses | Hoy |
| Todo | Sin filtro de fecha (solo roles CEO, ADMINISTRADOR y COO en colecciones sin límite) | — |

El resumen muestra el nombre del rango y los días reales (ej. `01/10 – 21/10/2026`). El rango se envía al backend en `fechasFiltro` con el formato `YYYYMMDD-YYYYMMDD`.

---

## Límites de exportación

Solo aplican a las colecciones con límite: **`PedidosInter`** y **`LiquidacionesLogigho`**. Los máximos salen de mediciones en el navegador con Pedidos (60 columnas).

| Formato | Nivel | Registros máximos | Otra regla |
| --- | --- | --- | --- |
| CSV | ALTO | 300.000 | — |
| Excel | MEDIO | 130.000 | — |
| JSON | MEDIO | 150.000 | — |
| PDF | BAJO | 150.000 | Máximo **10 columnas** (obliga "Seleccionar columnas específicas") |

Cómo se aplica:

1. Al elegir la colección se cuentan los registros de cada rango con las tiendas del usuario (`contarRangos`). Un rango solo se puede elegir si sus registros caben en el formato (`rangoPermitido`); "Última semana" siempre se puede. Si el conteo falla, se usan meses fijos por formato (Excel 1, JSON 3, PDF 3, CSV 6).
2. Las **fechas personalizadas** pueden durar como máximo lo del rango más largo que cabe. En el calendario los días que se pasan salen **tachados**; en celular el calendario del teléfono usa `min`/`max` y, si el teléfono no lo respeta, avisa "Fecha fuera del límite".
3. Al cambiar de formato se ajusta solo: si el rango ya no cabe pasa a "Este mes" (o "Última semana") y en PDF deja máximo 10 columnas.
4. Al exportar, `validarLimiteExportacion()` revisa todo de nuevo. Con fechas personalizadas vuelve a contar en el backend, porque un periodo viejo puede tener más registros.
5. Si se pasa, el aviso dice hasta dónde se puede y sugiere escribirle al equipo de desarrollo.

En las colecciones sin límite el máximo del formato se usa solo como referencia en el panel (no bloquea).

---

## Panel "Registros a exportar"

| Dato | Cómo se calcula |
| --- | --- |
| Registros | `registrosEstimados`: el conteo del rango si lo describe (colección con límite, rango de los botones y sin filtros); si no, `conteoSeleccion` (consulta con fechas + filtros). Con **Ecosistema** primero se cambia por sus tiendas y se cuenta igual; si no queda ninguna tienda el total es 0 sin consultar |
| Barra y porcentaje | Registros ÷ máximo del formato. Verde por debajo del 70%, naranja hasta el 100%, roja si se pasa. Con pocos registros dice "Menos del 1%" y deja un trazo visible |
| Columnas | Columnas que lleva el archivo |
| Celdas | Registros × columnas |
| Aún caben | Máximo del formato − registros |
| Estado | "Listo para exportar" o lo que falta: filtro sin agregar, elegir al menos una columna, de 1 a 10 columnas en PDF, o "Te pasas del máximo: acorta las fechas" |

---

## Búsqueda por guía o preenvío

Los campos con control `NUMBER` que no son `DECIMAL` (ej. Numero Guia, IdTienda) aceptan varios valores:

- Se separan por coma, punto y coma, tabulación o salto de línea. Al **pegar una columna de Excel** los saltos de línea se convierten en comas (`pegarVariosValores`). El pegado se hace con `document.execCommand('insertText')` para que **Ctrl+Z** lo deshaga.
- Al agregar se quitan espacios, vacíos y repetidos. Si algún valor no es número, avisa "Valor no válido" y no agrega el filtro.
- Máximo **400 valores** por filtro. Toda la configuración viaja en la URL de la pestaña de `export-runner`; en las pruebas más de 600 guías dieron error HTTP 431.
- Mientras se escribe se ve cuántos valores lleva, y en los filtros aplicados dice "N guías" o "N preenvíos".
- **Enter** en el Valor agrega el filtro (`keydown`, no `keyup`: así el Enter que eligió el campo no guarda el filtro sin querer).

### Revisión de guías y preenvíos que no existen

Si el campo es de guía o de preenvío (`tipoIdentificador`: el nombre contiene "guia" o "preenvio", sin importar tildes) y el valor son solo números separados por coma, antes de agregarlo se revisa en el backend cuáles existen (`revisarIdentificadoresExistentes`):

1. Se consulta solo ese campo con los números, el `FiltroReferencia` de la colección (tiendas del usuario) y **sin fechas**, con `lote=8000` porque una guía puede tener varios registros (ej. en Liquidaciones). Mientras tanto el botón dice "Revisando…".
2. Los números largos llegan como `{ $numberLong }` o como `"240063242200.0"`; `textoDeValor` los deja como texto para compararlos.
3. Si faltan algunos, pregunta "N guías no se encontraron" (no existen o no son de tus tiendas) con la lista (máximo 20) y los botones **Agregar solo las N que sí existen** o **Corregir** (deja el texto para arreglarlo).
4. Si no existe ninguno, solo deja **Corregir**.
5. Si la consulta falla, se agregan todos sin revisar (la exportación funciona igual).

Los avisos usan "guía/guías" o "preenvío/preenvíos" según el campo.

### Sin rango ni límite

Cuando hay un filtro por guía **o por preenvío** (`filtroPorGuias`) se exporta **sin rango de fechas** (`fechasFiltro=todo`) y **sin límite de registros**, porque son pocas filas y el rango las dejaría por fuera. El backend recibe los valores separados por coma y los busca con `$in`. Los textos del resumen y del aviso de fechas dicen por qué se busca (`textoBusquedaPorNumero`: "guía", "preenvío" o "guía y preenvío") y cuántos son (`resumenBusquedaPorNumero`, ej. "3 guías · 1 preenvío").

---

## Filtro por Ecosistema

En `PedidosInter` y `LiquidacionesLogigho` se agrega un filtro **virtual** llamado `Ecosistema` (no es una columna de la colección y no se exporta). Es `MULTISELECT`: se pueden marcar **varios ecosistemas**.

1. Las opciones salen de la colección `Tienda` (`NombreTienda`, `Ecosistema`). Los ecosistemas se agrupan sin espacios ni mayúsculas (ej. "PRUEBAS", "PRUEBAS " y "Pruebas" quedan juntos), se descartan los vacíos y **solo salen los que tienen tiendas del usuario**.
2. `tiendasDeEcosistemas()` junta las tiendas de los ecosistemas marcados sin repetir, deja solo las asignadas al usuario (salvo "Todas") y, si además hay filtro Tienda, solo las elegidas ahí.
3. Al exportar, `resolverFiltrosParaEnvio()` cambia `Ecosistema` por `Tienda=a,b,c` con esas tiendas. Si no queda ninguna avisa "Sin tiendas para exportar" y no exporta.
4. El conteo de registros del panel también funciona con Ecosistema (se cuenta con sus tiendas).

### Largo de la URL

Las tiendas viajan en la URL de `export-runner`. Para que no pase el error 431:

- El máximo es `maxLargoFiltrosUrl` = **7000 caracteres** de filtros (600 guías ≈ 9.000 pasaron; se deja margen para el token). Si las tiendas asignadas del usuario ya ocupan más, se usa ese largo, porque hoy ya le viajan todas por `#VALOR` y no se le debe bloquear nada.
- Al marcar un ecosistema que ya no cabe, avisa **"Ya no puedes agregar más ecosistemas"** y no lo marca. En la práctica solo pasa con usuarios que tienen "Todas".
- Al exportar se revisa otra vez (por si se agregaron guías después) y, si no cabe, avisa **"Demasiadas tiendas"**: quitar un ecosistema o algunas guías.

---

## Permisos por rol

| Dónde | Campo | Efecto |
| --- | --- | --- |
| `ListaColecciones` | `Roles` (lista o texto separado por comas) | Sin `Roles` la ven todos; con `Roles` solo los usuarios que tengan alguno |
| `ConfiguracionCamposColeccion` | `Roles` | Igual, pero por campo: si el usuario no tiene el rol, el campo no sale en columnas ni filtros y no se exporta |
| Rango "Todo" | — | Solo CEO, ADMINISTRADOR y COO (colecciones sin límite) |

Los roles se leen de `sessionStorage['roles_asignados']` y se comparan sin distinguir mayúsculas.

---

## Mensaje al equipo de desarrollo

En el aviso del límite de fechas hay un campo para escribirle al equipo de desarrollo cuando el usuario necesita exportar más de lo permitido:

- Se envía con `NotificacionesService.enviar('SOLICITUD_EXPORTACION', mensaje, contexto)` y le llega a la [campana](../../../components/campana-notificaciones.md) de los usuarios con rol `Desarrollador`.
- El contexto lleva lo que el usuario intentaba exportar: colección, formato, rango (`dd/mm/aaaa a dd/mm/aaaa`), registros, columnas y límite. Así el equipo no tiene que preguntarle.
- Cuando el equipo responde, al usuario le llega a su campana una notificación "Tu solicitud fue atendida" con la respuesta.
- Máximo 500 caracteres (con contador) y un mensaje cada 2 minutos.
- Los avisos usan `AvisoNotificacionService`:

| Caso | Aviso |
| --- | --- |
| Enviado | Verde: "Mensaje enviado. El equipo de desarrollo ya lo tiene en sus notificaciones. Te avisaremos cuando responda." Se limpia el campo |
| Antes de 2 minutos | Azul: "Ya enviaste un mensaje. Podrás enviar otro en N min." |
| Falla | Rojo: "No se pudo enviar". Conserva el texto para reintentar |
| Escritura apagada (`EscrituraDesactivadaError`) | Azul: "Envío en preparación". Conserva el texto |

---

## Filtro elegido sin agregar

Si el usuario eligió campo y valor en Filtros avanzados pero no tocó **Agregar**:

- El resumen muestra un aviso amarillo y el estado dice "Tienes un filtro sin agregar".
- Al exportar pregunta: **Agregar y exportar**, **Exportar sin ese filtro** o **Cancelar**.

---

## Teclado en Filtros avanzados

| Dónde | Tecla | Resultado |
| --- | --- | --- |
| Campo | `↓` / `↑` | Abre la lista y mueve la opción marcada (`alTeclearCampo`, `indiceCampoActivo`) |
| Campo | `Enter` | Elige la marcada, o la única que queda al buscar, y lleva el cursor al Valor |
| Campo | `Enter` con un campo ya elegido y la lista cerrada | Lleva el cursor al Valor |
| Campo | `Escape` | Cierra la lista |
| Valor | `Enter` | Agrega el filtro |

La lista usa `role="combobox"` / `listbox` y la opción marcada se ve con `dropdown-item--activo`.

---

## Celular (hasta 576px)

| Parte | Comportamiento |
| --- | --- |
| Listas de Colección, Formato, Campo y Valor | Paneles desde abajo (`.hoja`) con barrita, título, ✕ y buscador; en multiselect botón "Listo (N)". `cerrarHojas()` cierra cualquiera abierta |
| Fechas | Calendario del teléfono (`type="date"`) con `min`/`max` según el límite (`alElegirFechaNativa`). Rangos en pastillas deslizables |
| Columnas | 2 por fila, con botones "Todas" (en PDF solo hasta 10) y "Ninguna" |
| Información de uso | Plegable, empieza cerrada |
| Encabezado | De la fila "Ir directo a" solo queda el botón del tour, a lo ancho |
| Campos y avisos | Campos de 48px de alto y letra de 16px (evita el zoom del teléfono). Los avisos Swal salen a lo ancho con los botones apilados |

---

## Tour guiado

Usa el componente compartido [Tour guiado](../../../components/guided-tour.md). Tiene 14 pasos en el orden de la pantalla (`pasosTour`):

| # | Id | Paso |
| --- | --- | --- |
| 1 | `ex-tour-coleccion` | Colección de datos |
| 2 | `ex-tour-ir-a` | Ir directo a (no sale en celular) |
| 3 | `ex-tour-formato` | Formato y su nivel |
| 4 | `ex-tour-columnas` | Columnas y buscador |
| 5 | `ex-tour-rangos` | Rangos de fecha (Este mes, Último mes…) |
| 6 | `ex-tour-limite` | Hasta dónde puedes exportar y mensaje al equipo |
| 7 | `ex-tour-fechas` | Fechas personalizadas |
| 8 | `ex-tour-filtros` | Filtros avanzados, varias guías y Ecosistema |
| 9 | `ex-tour-aplicados` | Filtros aplicados |
| 10 | `ex-tour-barra` | Así queda tu exportación |
| 11 | `ex-tour-registros` | Registros a exportar |
| 12 | `ex-tour-exportar` | Botón Exportar |
| 13 | `tour-campana-notificaciones` | La [campana](../../../components/campana-notificaciones.md) del header, donde llega la respuesta del equipo |
| 14 | `ex-tour-btn` | Ver el tour de nuevo |

`armarPasosTour()` quita el paso de "Ir directo a" en celular (ahí esa fila solo muestra el botón del tour). Los pasos que no están en pantalla (según colección, formato o rol) se saltan solos.

---

## Cómo funciona el filtro de tiendas asignadas

El usuario tiene en `sessionStorage['tiendas_asignadas']` los nombres de tienda a los que tiene acceso, separados por coma (ej: `"BENDITA FRAGANCIA,BTK SHOR DAMA"`). Si contiene `"Todas"`, ve todas sin restricción.

Al cargar las opciones de un campo `COLECCION` que referencia `Tienda`, el componente:
1. Trae todas las tiendas del backend sin filtro de nombre (solo `Estado=ACTIVO`)
2. Construye los options usando `NombreTienda` como `value` (no el `Id` numérico) para que el filtro enviado al backend sea el nombre. En los campos de Id (ej. `IdTienda`) se deja el número
3. Filtra en cliente comparando el `label` (NombreTienda) contra la lista asignada, **solo en el campo de la tienda dueña del registro** (los que tienen `#VALOR` en `FiltroReferencia`, o `Tienda`/`NombreTienda`, más `IdTienda`). Campos como `TiendaDestino` o `TiendaReferida` muestran todas

> **Por qué se filtra en cliente:** `ConsumoGenericoService.consultarGenerico()` usa `HttpParams.set()` internamente, que sobreescribe valores duplicados. Enviar `Tienda=A&Tienda=B` vía ese servicio no es posible. La alternativa de enviarlo en el querystring directo codifica los espacios como `%20` y el backend real no los trata como OR múltiple.

---

## Pruebas

`export-data.component.spec.ts` (78 pruebas) cubre: aviso sin colección, rangos Este mes / Último mes, buscador de columnas, filtro con varias guías, revisión de guías y preenvíos que no existen, búsqueda por guía o preenvío sin rango ni límite, varios ecosistemas y largo de la URL, filtro sin agregar, teclado en Filtros avanzados, panel del resumen, lista de formatos, pastilla "Ir a exportar", celular, accesos "Ir directo a", tour y mensaje al equipo de desarrollo.

`directives/ayuda-flotante.directive.spec.ts`: 5 pruebas ([ver directiva](directives/ayuda-flotante-directive.md)).

```
npx ng test --watch=false --browsers=ChromeHeadless --include=src/app/views/export-data/export-data.component.spec.ts
```

---

## Historial de cambios

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-05-13 | Adalberto González | Documentación inicial |
| 2026-05-13 | Adalberto González | Fix desfase UTC: parseo de `YYYY-MM-DD` como fecha local para evitar desfase de un día |
| 2026-05-13 | Adalberto González | Fix filtrado de tiendas asignadas: aplica a todas las colecciones por `CampoEtiquetaReferencia === 'NombreTienda'`, no solo a las que tienen `#VALOR` en `FiltroReferencia` |
| 2026-05-13 | Adalberto González | Fix `CampoValorReferencia`: se usa `NombreTienda` como value en opciones de Tienda para que el backend reciba el nombre y no el Id numérico |
| 2026-05-13 | Adalberto González | Eliminado botón "Ver Ejemplos" y su panel de filtros anidados |
| 2026-10-16 | Yadilton Torres | Filtro virtual Ecosistema en Pedidos y Liquidaciones (se envía como `Tienda=a,b,c`) |
| 2026-10-16 | Yadilton Torres | Permisos por rol en colecciones (`ListaColecciones.Roles`) y en campos (`ConfiguracionCamposColeccion.Roles`) |
| 2026-10-16 | Yadilton Torres | Límites de exportación por formato en `PedidosInter` y `LiquidacionesLogigho`: conteo por rango, días tachados y máximo 10 columnas en PDF |
| 2026-10-20 | Yadilton Torres | Rangos "Este mes" (día 1 a hoy) y "Último mes" (mes anterior completo); "Este mes" por defecto |
| 2026-10-20 | Yadilton Torres | Buscador de columnas y lista de formatos con niveles ALTO / MEDIO / BAJO |
| 2026-10-20 | Yadilton Torres | Varias guías en un filtro (separadas por coma o pegadas de Excel); por guía se exporta sin rango ni límite |
| 2026-10-20 | Yadilton Torres | Resumen "Así queda tu exportación", accesos "Ir directo a", pastilla "Ir a exportar", aviso sin colección y aviso de filtro sin agregar |
| 2026-10-20 | Yadilton Torres | Diseño para celular: paneles desde abajo, calendario del teléfono, columnas en 2 por fila e información de uso plegable |
| 2026-10-20 | Yadilton Torres | Fix `limpiarLocalStorageAntiguo`: ahora lee bien la hora del `exportId` (antes nunca borraba) y borra a los 30 minutos |
| 2026-10-21 | Yadilton Torres | Eliminado el botón "Exportar Servidor" (`exportarDatosServidor`, `puedeVerExportarServidor`) porque el backend no responde (504) |
| 2026-10-21 | Yadilton Torres | Panel "Registros a exportar" en todas las colecciones (también sin límite y con filtros) con barra, porcentaje y fichas Columnas / Celdas / Aún caben |
| 2026-10-21 | Yadilton Torres | Máximo de 400 valores por filtro y mensaje al equipo de desarrollo como notificación de la campana |
| 2026-10-21 | Yadilton Torres | Avisos del mensaje al equipo con `AvisoNotificacionService` (antes Swal) y respuesta del equipo en la campana de quien lo envió |
| 2026-10-21 | Yadilton Torres | Varios ecosistemas a la vez (`MULTISELECT`), solo los que tienen tiendas del usuario, control del largo de la URL y conteo de registros con Ecosistema |
| 2026-10-21 | Yadilton Torres | Revisión de guías y preenvíos que no existen al agregarlos ("Agregar solo las que sí existen" / "Corregir"); el preenvío también exporta sin rango ni límite |
| 2026-10-21 | Yadilton Torres | Ayudas al pasar el mouse con la directiva `appAyuda` (reemplaza los `title`); los rangos muestran sus fechas exactas |
| 2026-10-21 | Yadilton Torres | Teclado en el Campo de Filtros avanzados, Enter en Valor agrega, Ctrl+Z deshace el pegado de guías y botones de filtros con colores nuevos |
| 2026-10-21 | Yadilton Torres | Tour de 14 pasos en el orden de la pantalla (Ir directo a, Registros a exportar, Exportar y campana de notificaciones) |

---

## Observaciones

- **`ConfiguracionCamposColeccion` es la fuente de verdad** del módulo. Si un campo no aparece, aparece con el control equivocado o el selector de tiendas sale vacío, el primer lugar a revisar es esa colección en MongoDB. Los campos con `Estado: INACTIVO` o sin permiso para el rol del usuario son ignorados por completo (tampoco se exportan).
- **Campos DATE están desactivados** (`Estado: INACTIVO` en `ConfiguracionCamposColeccion`) en todas las colecciones porque el filtro avanzado de igualdad exacta no hace match con campos que guardan datetime completo (`"2026-02-16 18:06:01"`). El filtro de rango principal (`fechasFiltro`) sí funciona correctamente porque usa el sistema propio del backend.
- **Los límites son solo del frontend.** Lo que se oculta por rol o por límite se sigue pudiendo pedir directo al backend; la validación real debe estar en el servidor.
- **Mediciones que dieron los límites** (Pedidos, 60 columnas, navegador): Excel falló desde ~116 mil registros (`XLSX.write`), JSON falló con ~439 mil, PDF con 60 columnas falló desde ~16 mil y con 10 columnas llegó a 187 mil. El backend da 504/500 desde ~290 mil registros. El límite es por conteo (no por rol) porque hay usuarios con ~300 tiendas.
- **Tope de 400 guías y 7000 caracteres de filtros:** todo viaja en la URL de `export-runner` (filtros, tiendas y token). Si algún día se necesitan más, hay que pasar los filtros por `localStorage` y consultar por partes.
- **La revisión de guías es solo una ayuda:** si el backend no responde, el filtro se agrega sin revisar.
