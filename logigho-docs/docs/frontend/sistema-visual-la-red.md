---

## Autor: Iker Acevedo (decisión) · redactado con Claude

Fecha creacion: 2026-11-23
Estado: desarrollo (verificado en local; pendiente de validar en PreProd)
Tipo: estándar de estilos

# Sistema visual «La Red»: tokens, utilidades y cómo migrar una vista

**Ubicación del código:** `src/scss/lared/` y `src/scss/_lared-tema.scss` (repo `Angular SitioLogiGho`, rama `feature/rediseno-integral-la-red`)

!!! warning "Estado: verificado en local, pendiente de PreProd"
    Descrito sobre la cabeza de la rama al 2026-11-23. Cifras de verificación y pendientes: [Rediseño «La Red»](tema-la-red.md). Marcas: **observado** (archivo citado), **Iker (fecha)**, **hipótesis**.

!!! info "No es el estándar de «Experiencias»"
    [Sistema de Experiencias: tokens y estilos](experiencias/tokens-y-estilos.md) es una **propuesta** (no aprobada) con el mismo prefijo `--lg-` pero otro mecanismo (`data-experiencia`) y otros nombres (`--lg-color-primario`, `--lg-fondo`…). Lo que existe en el código es lo que describe esta página. Algunos nombres coinciden (`--lg-texto`, `--lg-superficie`, `--lg-borde`, `--lg-peligro`, `--lg-radio-md`) y sus valores no están reconciliados: si la propuesta se aprueba habrá que alinearlos (**observado** al comparar ambos documentos con `_tokens.scss`).

## Cómo se activa

```html
<!-- src/index.html -->
<html lang="es" data-lg-tema="la-red">
```

- El atributo `data-lg-tema="la-red"` en `<html>` define todos los tokens `--lg-*`. Las vistas migradas **consumen esos tokens y no funcionan sin él**: por eso quitar el atributo **no** es una forma válida de revertir (ver [Rollback](../guias/rediseno-la-red-rollback-y-hotfix.md)).
- `src/scss/styles.scss` importa `_lared-tema.scss`, que carga en este orden: `tokens` → `coreui` → `utilidades` → `overlays` → `movimiento` → `mesa` → `pendientes` → `basico` → `impresion` (**observado**).
- Las fuentes (Plus Jakarta Sans 600/700 e Inter 400/500/600) se piden a Google Fonts desde `index.html`; antes solo llegaban por el `@import` de `registro.scss` y las rutas privadas quedaban en Roboto.

| Archivo (`src/scss/lared/`) | Qué contiene |
|---|---|
| `_tokens.scss` | Tokens `--lg-*` de noche, día y claro; familia `--lg-noche-*` y `--lg-shell-*` del shell |
| `_coreui.scss` | Remapeo de variables de CoreUI 5 (`.card`, `.table`, `.btn-*`, `.form-control`, `.modal`, `.dropdown-menu`, `.nav-tabs`, `.alert-*`, `.pagination`) |
| `_utilidades.scss` | Clases `lg-*` (tarjeta, tabla, badge, botones, campo, KPI, `lg-isla-clara`) |
| `_mixins.scss` | Los mismos estilos como mixins para el SCSS de componente (no emite CSS) |
| `_overlays.scss` | SweetAlert2 y calendario de ng-bootstrap, que se montan fuera del componente |
| `_movimiento.scss` | `@keyframes lg-*` y la coreografía de entrada (ver [Movimiento](movimiento-y-navegacion-la-red.md)) |
| `_mesa.scss` | Mesa clara de lo no migrado y paso a noche con `:has()` |
| `_basico.scss` | Modo básico y teléfono (ver [Modo básico](modo-basico-y-movil-la-red.md)) |
| `_impresion.scss` | Tokens de día al imprimir |
| `_pendientes.scss` | Generado desde la matriz de pantallas; hoy sin componentes pendientes |

## Noche, claro y las clases `lg-vista-noche`, `lg-noche`, `lg-dia`

| Clase o atributo | Dónde va | Qué hace |
|---|---|---|
| (sin atributo `data-lg-modo`) | `<html>` | Noche. Es el modo por defecto (Iker, 2026-11-22) |
| `data-lg-modo="dia"` | `<html>` | Modo claro «Papel de ruta» ([Modo claro](modo-claro-la-red.md)) |
| `lg-vista-noche` | Raíz de la plantilla de una **vista enrutada** ya migrada | Marca «esta vista está migrada». `_mesa.scss` lo detecta con `:has()` desde el contenedor del layout y le da los tokens de noche (o de claro si el modo es claro) |
| `lg-noche` | Raíz visible de un **componente compartido no enrutado** (modal, importador) | Lo mismo, solo para ese componente |
| `lg-dia` | Un bloque | Devuelve los tokens de día dentro del elemento |
| `lg-isla-clara` | Junto con `lg-dia` | Isla clara para contenido de terceros que no se tematiza desde CSS: las gráficas de Chart.js con ejes y series definidos en TypeScript. 4 usos en plantillas (observado) |

!!! note "El nombre engaña"
    `lg-vista-noche` y `lg-noche` significan «migrada al tema», no «oscura». Con `data-lg-modo="dia"` esas mismas clases reciben los tokens claros. Fuera del shell privado (landing, páginas públicas) los tokens valen día salvo que el elemento lleve `lg-noche`.

**Una vista no migrada nunca queda ilegible:** el contenedor de la vista (`.wrapper .body > .container`) vale «mesa clara» por defecto. En un navegador sin `:has()` una vista migrada también se queda en mesa clara (degrada a legible, no a roto).

## Tokens `--lg-*`

Fuente: `_tokens.scss`. Los valores de noche y claro son distintos; el código de una vista **nunca** escribe colores: usa el token.

| Familia | Tokens principales |
|---|---|
| Fondo y superficies | `--lg-pagina` (noche `#0a1f33`), `--lg-superficie` (`#102f4b`), `--lg-superficie-2`, `--lg-superficie-3`, `--lg-fila-hover`, `--lg-fila-sel`, `--lg-fila-sel-borde` |
| Texto | `--lg-tinta` (títulos), `--lg-texto`, `--lg-texto-2`, `--lg-texto-3`, `--lg-texto-off`, `--lg-placeholder` |
| Bordes y foco | `--lg-borde`, `--lg-borde-control`, `--lg-foco`, `--lg-anillo` |
| Enlaces y acento | `--lg-azul`, `--lg-azul-hover`, `--lg-acento`, `--lg-peligro`, `--lg-peligro-solido` |
| Estados | `--lg-{exito\|alerta\|error\|info\|neutro}-{texto\|fondo\|borde\|solido}` |
| Acción principal | `--lg-primario` (degradado `#c2410c → #c41e2a`), `--lg-primario-hover`, `--lg-primario-plano`, `--lg-primario-texto`, `--lg-primario-sombra`; secundario: `--lg-secundario-fondo`, `--lg-secundario-hover` |
| Marca cálida | `--lg-calido-1`, `--lg-calido-2`, `--lg-linea-calida`, `--lg-linea-calida-v`, `--lg-linea-alto` (3 px) |
| Tipografía | `--lg-font-titulo`, `--lg-font-cuerpo`, `--lg-cifras` (cifras tabulares), `--lg-txt-2xs` … `--lg-txt-kpi` |
| Espacio y forma | `--lg-esp-1` … `--lg-esp-10`, `--lg-gutter`, `--lg-radio-sm\|md\|lg\|pill` (6/10/14 px), `--lg-fila-alto` (40 px), `--lg-control-alto` (36 px), `--lg-elev-1\|2\|3` |
| Movimiento | `--lg-ease-salida`, `--lg-ease-cerrar`, `--lg-ease-rebote`, `--lg-dur-1\|2\|3` (120/160/240 ms), `--lg-dur-linea` (420 ms), `--lg-escalon` (30 ms), `--lg-dist-entra`, `--lg-escala-entra` |
| Capas | `--lg-z-modal` (1060), `--lg-z-popover` (1070), `--lg-z-toast` (1080) |
| Esqueleto y telón | `--lg-esqueleto-a\|b`, `--lg-telon` |
| Solo shell | `--lg-noche-*` (constantes de noche) y `--lg-shell-*` (cabecera, menú lateral, buscador, campana, menú móvil). **No se usan en vistas** |

Decisiones de diseño relevantes (**Iker, 2026-11-22** salvo que se indique): acción principal en degradado cálido **oscurecido** (el del login no llega a contraste AA con texto pequeño blanco); densidad operativa (fila 40 px, control 36 px, cuerpo 14 px, tabla 13 px); estados siempre con texto o icono además del color.

## Remapeo de CoreUI

`_coreui.scss` redeclara las variables `--cui-*` de cada componente de CoreUI sobre los tokens. Solo el prefijo del tema va dentro de `:where()`: así cada regla tiene la misma especificidad que la de CoreUI y le gana por orden, pero **pierde contra el estilo de cada componente Angular** (decisión D13). Por eso las vistas ya migradas que traen su propio SCSS mandan sobre el tema.

## Mixins `lg.*` (SCSS de componente)

Para vistas donde no se pueden cambiar las clases de la plantilla. Se importan con la ruta relativa al archivo:

```scss
@use '../../../scss/lared/mixins' as lg;   // primera línea del archivo

.mi-boton { @include lg.btn-primario; }
.mi-tabla { @include lg.tabla; }
```

`btn-primario`, `btn-secundario`, `btn-peligro`, `btn-fantasma`, `btn-icono`, `control`, `etiqueta`, `tarjeta`, `linea-calida`, `titulo-vista`, `titulo-seccion`, `kpi-valor`, `kpi-etiqueta`, `tabla`, `tabla-marco`, `cifra`, `badge($estado)`, `modal-telon`, `modal-panel($animar)`, `modal-titulo`, `modal-pie`, `barra-resumen`, `scroll-fino`.

- `lg.modal-panel` anima la entrada: úsalo tal cual solo en paneles que se crean al abrir (`@if`). Si el modal vive siempre en el DOM y se muestra con una clase, usa `lg.modal-panel($animar: false)`.
- `@use` va en la **primera línea** del SCSS; si no, el build se rompe.

## Utilidades `lg-*` (plantillas)

`lg-tarjeta`, `lg-tarjeta--calida`, `lg-linea-calida`, `lg-tabla-marco`, `lg-tabla`, `lg-tabla--compacta`, `lg-num`, `lg-col-acciones`, `lg-badge` (`--exito`, `--alerta`, `--error`, `--info`), `lg-btn` (`--primario`, `--secundario`, `--peligro`, `--fantasma`), `lg-campo` (con `__etiqueta`, `__ayuda`, `__error`), `lg-control`, `lg-form`, `lg-barra-filtros`, `lg-kpi__etiqueta`, `lg-kpi__valor`, `lg-titulo`, `lg-subtitulo`, `lg-texto-2`, `lg-texto-3`, `lg-isla-clara`. Más las de movimiento (`lg-escalonar`, `lg-aparece`, `lg-aparece-rapido`, `lg-sin-entrada`) descritas en [Movimiento](movimiento-y-navegacion-la-red.md).

## Primitivas de presentación

Tres componentes standalone nuevos en `src/app/components/`, sin datos ni peticiones:

| Componente | Selector | Qué hace |
|---|---|---|
| `LgEstadoComponent` | `app-lg-estado` | Estados `vacio`, `error`, `sin-permiso` y `cargando` (esqueleto + texto oculto para lectores de pantalla), con acción opcional. `role="status"` o `"alert"` |
| `LgEncabezadoVistaComponent` | `app-lg-encabezado-vista` | Kicker, `<h1>`, descripción, acciones a la derecha y filtros debajo |
| `LgModalComponent` | `app-lg-modal` | Telón, panel con línea cálida, foco atrapado, Escape, bloqueo del scroll de `body` y pila de modales |

**Hoy ninguna vista las importa** (**observado**: búsqueda en `src/app` sin usos fuera de sus carpetas). Quedan disponibles para adoptarlas al tocar cada vista (M-370). Sus specs cubren 32 casos.

## Overlays de terceros, gráficas e impresión

- **SweetAlert2 se queda claro en los dos modos** (con línea cálida): hay 941 llamadas y 79 pasan `html:` con colores en línea pensados para fondo blanco; tematizarlo en noche exigiría revisarlas una a una (decisión D18; M-358). Sus botones conservan el color que cada llamada fija (los rojos significan peligro).
- **Calendario de ng-bootstrap** (`ngb-datepicker`): tematizado en `_overlays.scss`.
- **Gráficas de Chart.js** con colores en TypeScript: van sobre `lg-dia lg-isla-clara` (sobre noche los ejes quedaban a 2,39:1 de contraste). Mejora pendiente: leer tokens con `getComputedStyle` (M-356).
- **Impresión** (`@media print`): el contenido vuelve a tokens de día con fondo blanco, en noche y en claro, para no sacar texto claro sobre papel blanco (`_impresion.scss`).

## Cómo migrar o crear una vista

Resumen de `receta-slice.md` y `receta-barrido-claro.md` (carpeta de la spec del workspace, donde está el detalle completo).

### Qué se puede tocar

1. **`*.scss` de la vista** o componente: sustituir cada color fijo por su token y los botones, controles y tablas por mixins o utilidades.
2. **En la plantilla, solo clases:** añadir `lg-vista-noche` al elemento raíz (vista enrutada) o `lg-noche` (componente compartido). Se **añade**; no se quita ni se cambia ninguna otra clase ni se añade un contenedor.
3. **Colores para el modo claro**, con una de dos formas (ver [Modo claro](modo-claro-la-red.md)).

### Qué no se puede tocar

- Handlers `(click)`, `(ngSubmit)`, `[(ngModel)]`, `formControlName`, `[disabled]`, `*ngIf`/`@if`, `*ngFor`/`@for`, `trackBy`, pipes, textos, `id`, `name`, `for`, `data-*`, `type` de botones, `routerLink`, `href`.
- Añadir o quitar `@if` alrededor de componentes hijos (cambia cuántas peticiones se hacen al montar).
- Meter o sacar un botón de un `<form>`, añadir un `<form>`, cambiar el orden de lo enfocable.
- Renombrar o quitar clases que el TypeScript, un spec o un padre consulten (buscar el nombre en `*.ts` antes). En la duda, añadir la clase nueva y conservar la antigua.
- Cambiar textos visibles, formatos de moneda o fecha, columnas, orden o alineación de cifras.
- TypeScript de producción en las vistas: solo SCSS y clases. (El TypeScript nuevo del rediseño está solo en el armazón; ver [ADR-RED-003](../arquitectura/adr/rediseno-la-red/ADR-RED-003-typescript-nuevo-solo-en-el-armazon.md).)

### Reglas de estilo

- Colores, radios, sombras, tipografía y movimiento salen de tokens. **Prohibido hex nuevos** fuera de `src/scss/lared/` (excepción: `#fff` sobre un color sólido de marca o estado). Nunca `--lg-noche-*` en una vista.
- **Una acción principal por vista o modal.** Las acciones de dinero o destructivas (desembolsar, retirar, pagar, aprobar, anular, eliminar) conservan su peso relativo de la base: una de dinero y una destructiva contiguas no comparten tono.
- Foco visible: prohibido `outline: none` sin sustituto (`outline: 2px solid var(--lg-foco)`).
- Teléfono (390 px): sin scroll horizontal de página; las tablas anchas se desplazan dentro de su contenedor.
- Hover solo con puntero fino (`@media (hover: hover) and (pointer: fine)`) y sin transiciones de hover de más de 160 ms.
- Animaciones con `backwards`, solo `transform` y `opacity`; los indicadores de carga llevan un nombre reconocible (`spinner`, `girando`, `cargando`, `loading`, `shimmer`) o la clase `lg-indicador` para que el [modo básico](modo-basico-y-movil-la-red.md) no los congele.
- No usar reglas de elemento sin clase (`button { }`) ni fijar `display`, `position`, tamaños o sombras sobre hosts ajenos. Un hijo montado con su modal cerrado que abra huecos se resuelve con `display: contents` en el SCSS del padre, nunca con `@if`.
- Pantallas públicas: el tema remapea `--cui-heading-color`; una pantalla pública con títulos que heredan color debe dejarla en `initial` (así se rompió y se arregló «Recupera tu contraseña»).
- Los tokens interiores **no importan nada** de `views/pages/registro/la-red` (arrastraría Three.js al bundle interior).

### Cómo comprobarlo

Con las herramientas del workspace (`tools/rediseno-la-red/`, no navegables desde este sitio): capturar la ruta a 1440 y 390 px, verificar la paridad de peticiones (`paridad.js`, debe decir `0 diferencias`) y, si el SCSS tocó noche, comparar contra la línea base. `herramientas/verificacion-final.sh verificar` repite toda la verificación con un comando (red bloqueada; no toca servicios). `tools/rediseno-la-red/explorar.js` (abre la plataforma con datos sintéticos) tiene comprobada solo la sintaxis: **no se ha ejecutado con ventana** (`ENTREGA.md` §6).

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-11-23 | Iker Acevedo · redactado con Claude | Primera versión, a partir de `src/scss/lared/` y de la spec 021. |
