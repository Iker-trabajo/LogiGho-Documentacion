---

## Autor: Iker Acevedo (decisión) · redactado con Claude

Fecha creacion: 2026-11-23
Estado: desarrollo (verificado en local; pendiente de validar en PreProd)
Tipo: componente

# Movimiento y navegación entre módulos

**Directiva:** `lgNavegacion` (`src/app/layout/default-layout/directives/lg-navegacion.directive.ts`)  
**Precarga:** `PrecargaLaRed` (`src/app/layout/default-layout/precarga/precarga-la-red.ts`)  
**Estilos:** `src/scss/lared/_movimiento.scss`  
**Acceso:** Autenticado (shell privado)

!!! warning "Estado: verificado en local, pendiente de PreProd"
    Cifras y pendientes en [Rediseño «La Red»](tema-la-red.md). Sin probar en navegador: desplegables de CoreUI sobre Popper, `:active` de botones y el barrido de la línea en SweetAlert. Sin probar en iPhone real: el movimiento usa `translate` y `scale` como propiedades sueltas y `:has()`. Marcas: **observado** (archivo citado), **Iker (fecha)**, **hipótesis**.

---

## ¿Qué hace?

Da vida a las pantallas sin hacerlas esperar: cada vista entra por partes al abrirse, los modales y desplegables aparecen con suavidad y, al cambiar de módulo, el menú responde al instante mientras la pantalla nueva se prepara. Iker pidió una transición «lo más llamativa posible» al cambiar de módulo (decisión D43); la sensación llamativa está en la línea cálida (la firma del login) y en la coreografía, no en la duración.

En **teléfono, tableta, cualquier dispositivo táctil y [modo básico](modo-basico-y-movil-la-red.md) no hay animaciones**.

---

## Reglas de movimiento

| Regla | Detalle |
|---|---|
| Solo composición | `transform`, `opacity` y las propiedades sueltas `translate` y `scale`. Nada que repinte la página |
| `backwards`, no `both` | Con `both` Chromium deja una capa compuesta viva al terminar; con `backwards` no. Excepciones declaradas abajo |
| Nada infinito | El tema no tiene animaciones infinitas. Los indicadores de carga son la excepción heredada (ver abajo) |
| Una vez | Entradas solo al abrir un modal o montar una vista; nada por fila de tabla, ni en paginación, filtros o teclado |
| Movimiento reducido | `prefers-reduced-motion` conserva el fundido y quita el desplazamiento: `--lg-dist-entra: 0px`, `--lg-escala-entra: 1`, `--lg-dur-linea: 1ms`, `--lg-escalon: 0ms` |
| Curva y duraciones | `cubic-bezier(.22, 1, .36, 1)`; 120 ms (`--lg-dur-1`), 160 ms, 240 ms; la línea de modal 420 ms |
| Hover | Solo color o borde, sin desplazamiento, y solo con puntero fino |
| Sin View Transitions | Ver [ADR-RED-004](../arquitectura/adr/rediseno-la-red/ADR-RED-004-sin-view-transitions.md) |

**Excepciones declaradas** (**observado** en `_movimiento.scss`): la línea de progreso de la cabecera usa `both` porque su estado de espera (85 %) no es su estado natural; la salida del SweetAlert mantiene `both` y la del aviso flotante (`aviso-toast--salir`) usa `forwards`, porque su estado final es invisible hasta que se retiran; el esqueleto de carga pulsa 12 medias vueltas (≈ 16 s) y descansa en lugar de repetirse sin fin; el spinner del cargador de `index.html` (antes de que arranque Angular) sí es infinito.

---

## Catálogo

Los identificadores M1 a M10 vienen del diseño (`movimiento-fase-2.md` en la carpeta de la spec).

| # | Qué anima | Dónde se resuelve | Valores |
|---|---|---|---|
| M1 | **Entrada coreografiada de la vista:** encabezado → filtros o KPI → contenido | CSS global sobre `router-outlet + *` y los hijos de la raíz `lg-vista-noche` | Hijos 1 a 4 suben (`lg-sube`, `lg-sube-corto`) con huecos de 30/25/20 ms; del 5.º al 8.º solo se funden. Tope ≈ 275 ms y 8 elementos. Solo durante la ventana de la navegación: lo que aparece después no repite la entrada. En un cambio de **módulo** el primer bloque sube 12 px (`lg-sube-largo`) en vez de 8 |
| M2 | **Indicador de navegación:** atenuado del contenido saliente y línea cálida de la cabecera | Directiva `lgNavegacion` + CSS | Contenido a `opacity: .6` a los 80 ms de espera; la línea (`c-container::after` de la cabecera) se vacía a los 120 ms y avanza hacia el 85 % en 2,4 s (`lg-nav-avanza`); al llegar se completa. Con carga más rápida que el retardo no se ve nada salvo M1 |
| M3 | **Modales y hojas** | Mixin `lg.modal-panel` (≈ 80 usos), SweetAlert y `_coreui.scss` | Panel `lg-panel-entra` (sube 8 px y escala .985 → 1, 240 ms) y barrido de la línea cálida del panel (`lg-linea-entra`, 420 ms) |
| M4 | **Controles:** press y destino del menú | Global + shell | `.btn:active` escala .98; el enlace del menú pulsado se marca al instante con `lg-destino` |
| M5 | **Desplegables** (`.dropdown-menu.show` de CoreUI y el menú de columnas de la tabla) | `_coreui.scss` | `lg-desplegar`: opacidad, `scale: .97` y `translate`, 160 ms |
| M6 | **Avisos y SweetAlert tipo toast** | `_aviso-toast.scss`, `_overlays.scss` | Entrada con opacidad y `translateX` |
| M7 | **KPI escalonados** | Por vista, clase `lg-escalonar` en la fila (hija directa de la raíz) | Hasta 4 tarjetas, 200 ms, retardos de 30 ms; en 8 plantillas |
| M8 | **Contenido tras el esqueleto, vacío y error** | Por vista, clase `lg-aparece` | Solo opacidad (el contenido ocupa el sitio del esqueleto), 160 ms |
| M9 | **Cambio de pestaña** | Por vista, clase `lg-aparece-rapido` | Solo opacidad, 120 ms |
| M10 | **Precarga** | `PrecargaLaRed` | Ver abajo |

Utilidades para vistas: `lg-escalonar`, `lg-aparece`, `lg-aparece-rapido` y `lg-sin-entrada` (una raíz que ya tiene su propia entrada: sus hijos no se tocan). Se usan en unas 36 vistas según `ENTREGA.md` (conteo directo en plantillas: 31 archivos con alguna de las cuatro; no se reconcilió la diferencia, que puede deberse a clases usadas desde SCSS).

**Rechazados a propósito** (no se animan): filas de tabla, paginación, orden y filtros; el buscador del menú; cifras, saldos y gráficos; hover con elevación en tarjetas densas.

---

## Directiva `lgNavegacion`

Se coloca en el contenedor `.wrapper` del layout (`default-layout.component.html`). Refleja el estado del router en atributos para que el CSS dibuje la transición. **No decide rutas ni permisos y no toca datos.**

| Atributo o clase | Cuándo existe |
|---|---|
| `data-lg-nav="pendiente"` | Desde `NavigationStart` |
| `data-lg-nav="llega"` | 400 ms tras `NavigationEnd` |
| `data-lg-nav-tipo="modulo"` o `"vista"` | De `NavigationStart` a 700 ms tras `NavigationEnd`. `modulo` si cambia el primer segmento tras `/app`. No se pone si el destino es la misma ruta (recarga, o solo cambian query o fragmento) |
| `data-lg-entrada` | De `NavigationEnd` a +700 ms: es la ventana en la que las vistas nuevas hacen su entrada (M1) |
| Clase `lg-destino` | En el enlace del menú (escritorio o cajón del celular) cuya ruta coincide con el destino, de `NavigationStart` a `NavigationEnd`, `NavigationCancel` o `NavigationError` |

Casos que conviene saber:

- **Sin movimiento no escribe atributos.** En teléfono, tableta, táctil o con `data-lg-rendimiento` en `<html>` solo marca `lg-destino`. Motivo medido: cada cambio de esos atributos recalcula el estilo de todo el subárbol del `.wrapper` (15–21 ms en una vista de 245 nodos con CPU ×4) y en teléfono la animación está apagada, así que no compraba nada. Esa corrección bajó el coste de volver a un módulo en teléfono lento de +40–280 ms a +33–82 ms. En escritorio sigue habiendo un recálculo por atributo (M-384).
- `prefers-reduced-motion` **no** apaga la directiva: con esa preferencia el tema conserva los fundidos y necesita los atributos.
- Un fin de navegación sin inicio previo (primera entrada al layout tras el login) cuenta como `modulo` pero no pasa por «pendiente».
- Los `Cancel` y `Error` de una navegación ya reemplazada se ignoran. Los temporizadores corren fuera de `NgZone`.
- La consulta de «sin movimiento» (`(max-width: 768px), (pointer: coarse)`) debe coincidir en tres sitios: `_basico.scss`, esta directiva y `sidebar-interaccion.directive.ts` (esta última le suma `prefers-reduced-motion`). Si cambia una, cambian las tres.

---

## Precarga `PrecargaLaRed`

Estrategia de precarga de Angular registrada con `withPreloading(PrecargaLaRed)` en `src/app/app.config.ts`. **No hace ninguna petición a la API**: solo adelanta la descarga del código estático. Las guardas se siguen evaluando al navegar. Nunca se usa `PreloadAllModules` (serían ≈ 4,18 MB comprimidos en 286 archivos, cifra del diseño).

| Modo | Cuándo | Qué descarga |
|---|---|---|
| **Por intención** | `pointerenter` sostenido 65 ms (puntero fino) o `pointerdown` (táctil) sobre un enlace del menú | El `loadChildren` del área si falta y el `loadComponent` de esa vista, y nada más. Escucha delegada y pasiva en `document` (el menú lateral está fuera del `.wrapper`) |
| **En reposo** | A los 8 s del primer `NavigationEnd`, al registrar el menú del usuario y con el navegador libre (`requestIdleCallback`, respaldo de 3 s) | Solo las tablas de rutas (`loadChildren`) de las **áreas del menú del usuario**; ninguna vista. ≈ 11 KB comprimidos (`ENTREGA.md`) |
| **Corte por red** | Con ahorro de datos (`saveData`) o red `slow-2g`, `2g` o `3g` | Sin reposo y sin `pointerenter`; solo `pointerdown` |

- Una ruta con `canLoad` no se precarga (como Angular).
- Los 8 s de espera se añadieron tras medir que el reposo anterior (≈ 2,5 s) descargaba chunks a la vez que los datos de la primera vista y la retrasaba 0,3–0,4 s en red lenta. El diseño original hablaba de reposo «tras el primer `NavigationEnd`» con respaldo de 3 s: **manda el código** (8 s).
- El layout informa del menú con `precarga.registrarMenu(this.navItems)` una vez armado, con los módulos ya consultados: no hay peticiones nuevas.

---

## Qué se midió

Chromium sin interfaz, 5 repeticiones intercaladas, mediana (`ENTREGA.md` §5). No es un teléfono real ni Safari.

| Qué | Noche aprobada | Final | Lectura |
|---|---|---|---|
| Clic en el menú → primera respuesta visible, escritorio, primera visita | 48–147 ms | 19–31 ms | El enlace se marca y arranca la transición sin esperar al código del módulo |
| Toque → primera respuesta, teléfono lento, primera visita | 0,38–8,5 s | 0,04–0,28 s | Lo mismo, con CPU ×4 y red de 1,6 Mb/s |
| Vista lista, escritorio, primera visita | 48–147 ms | 53–151 ms | Sin diferencia fuera del ruido |
| Vista lista con el puntero 200 ms sobre el enlace antes del clic | 96–175 ms | 81–97 ms en 3 de 6 casos; igual en el resto | La precarga por intención adelanta la descarga (medido en el commit anterior al final) |
| Apertura de 3 modales | 28–78 ms | 25–77 ms | Sin diferencia; en «referidos», al reabrir, 4 cuadros de más de 25 ms frente a 1 |
| Cambio a un módulo ya visitado, teléfono lento | 95–277 ms | 128–359 ms | **+33 a +82 ms** (coste que queda) |

**Lo que no se promete resuelto:** el «congelado» al abrir algunos modales o vistas con datos reales es bloqueo del hilo principal dentro de la lógica de cada vista, que este rediseño no toca (decisión D44; M-382, M-386). La transición **tapa** la espera, no la elimina. Con tablas reales de miles de filas los costes por navegación serán mayores en los dos builds.

---

## Pruebas unitarias

| Archivo | Qué cubre |
|---|---|
| `lg-navegacion.directive.spec.ts` | Atributos, `lg-destino`, tipos de navegación, cancelaciones, sin movimiento (T01–T09 y casos añadidos) |
| `precarga-la-red.spec.ts` | Intención, reposo, corte por red, `canLoad`, sin peticiones (T10–T14); con los de la directiva suman 66 casos |
| `sidebar-interaccion.directive.spec.ts` | 3 casos nuevos: el plegado sin animación en básico y tableta |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-11-23 | Iker Acevedo · redactado con Claude | Primera versión de esta página. Commit `9b7c5adb` (posterior a la verificación): los botones «Acciones» y «Crear Pedidos» de Pedidos se expanden en 380 ms en lugar de 120 ms. |

---

## Observaciones

- Al volver de ancho de teléfono a escritorio (girar una tableta, redimensionar) las animaciones de entrada se reproducen una vez; es cosmético (M-383).
- La línea de progreso, con cargas de 120–300 ms, se vacía y se rellena muy rápido: decisión abierta de Iker.
- El menú del layout y su precarga dependen de que la ruta del enlace del menú sea interna; los enlaces con protocolo o de otro sitio se ignoran.
