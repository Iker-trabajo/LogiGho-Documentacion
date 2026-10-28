---
autor: Yadilton Torres
fecha_creacion: 2026-10-09
ultima_actualizacion: 2026-10-28
estado: desarrollo
nivel: 2
---

# Menú lateral (Default Layout)

**Autor:** Yadilton Torres

**Selector:** `app-dashboard`

**Ubicación:** `SitioLogiGho/src/app/layout/default-layout`

---

## ¿Qué hace?

Es la estructura que envuelve todas las pantallas internas de LogiGho: el **menú lateral** (sidebar), la [barra superior](default-header.md) (header), el contenido de la página (`router-outlet`), el botón de WhatsApp, la barra flotante y el footer.

El menú lateral se arma según los roles del usuario (colección `module`) y tiene:

- **Apertura sin parpadeo:** se abre solo si el mouse se queda encima un momento y tiene un margen antes de cerrarse.
- **Cierre al elegir una opción:** la barra se cierra sola aunque el mouse siga encima, espera a que la página nueva cargue para no dar tirones y los grupos abiertos se pliegan deslizándose.
- **Clics que no se pierden:** si el menú se mueve mientras se hace clic, la opción igual se abre.
- **Ubicación visible:** barrita azul en la sección abierta o actual, opción activa en morado y subgrupos con pastilla y línea de árbol.
- **Buscador del menú** con atajo **Ctrl + B** (**Cmd + B** en Mac), resultados agrupados por sección y botón **X** para borrar lo escrito.
- **Uso con el teclado:** con el menú abierto se recorre con las flechas, Enter abre submenús o va a la página y Escape cierra.
- **Barrita del scroll:** arrastrarla no cierra el menú aunque el mouse salga de la barra.
- **Celular:** el menú es un cajón completo que se cierra al elegir una opción o deslizando el dedo a la izquierda.

---

## Roles y acceso

| Acceso | Descripción |
|---|---|
| Autenticado | Requiere login. Es el layout de todas las rutas bajo `/app` con `AuthGuard` |
| Opciones del menú | Solo los módulos de la colección `module` que corresponden a los roles del usuario (ver [Rutas y navegación](../rutas.md)) |

---

## Estructura de archivos

```
default-layout/
├── default-layout.component.ts         → Arma el menú, valida acceso a módulos y hace la búsqueda
├── default-layout.component.html       → Sidebar, buscador (con la X para borrar), header, contenido y footer
├── default-layout.component.scss       → Solo importa los archivos de styles/ (10 líneas)
├── coreui-sidebar-ajuste.ts            → Ajuste a CoreUI: qué grupo del menú corresponde a la página
├── directives/
│   ├── sidebar-interaccion.directive.ts      → Mouse, clics, cierre al elegir, Ctrl + B, flechas, scroll y cajón del celular
│   └── sidebar-interaccion.directive.spec.ts → 33 pruebas del comportamiento del menú
├── styles/
│   ├── _variables.scss                 → Tiempos, curvas de animación y corte de celular
│   ├── _menu.scss                      → Colores, opciones, iconos, barrita azul, activo, grupos, subgrupos y marca del teclado
│   ├── _barra.scss                     → Ancho cerrada/abierta, apertura por capas y cierre sin que se note (clase "cerrando")
│   ├── _buscador.scss                  → Campo, etiqueta Ctrl B, botón X, "Sin resultados" y resultados agrupados
│   ├── _movil.scss                     → Cajón del celular
│   ├── _animaciones.scss               → Fundidos, cascadas, brillo del activo y "reducir movimiento"
│   └── _pagina.scss                    → Contenedor, cuerpo y botón de WhatsApp (ya existía)
└── default-header/                     → Barra superior (ver su página)

src/scss/
└── _animaciones-menu.scss              → @keyframes del menú (globales, ver Observaciones)
```

---

## Decoradores y propiedades clave

| Elemento | Nombre | Tipo | Descripción |
|---|---|---|---|
| `@ViewChild` | `sidebarInteraccion` | `SidebarInteraccionDirective` | Directiva del sidebar. El layout la usa para cerrar la barra cuando terminan de cargar los módulos |
| Propiedad | `navItems` | `INavData[]` | Menú completo, armado desde la colección `module` según los roles |
| Propiedad | `visibleNavItems` | `INavData[]` | Resultados del buscador: igual a `navItems` si no hay nada escrito |
| Propiedad | `searchTerm` | `string` | Texto escrito en el buscador del menú |

El `<c-sidebar-nav>` pinta `navItems` cuando la barra está cerrada (`interaccion.mostrarMenuCompleto`) y `visibleNavItems` cuando está abierta. Ver [Barra cerrada con búsqueda](#barra-cerrada-con-busqueda).

---

## Servicios y endpoints

| Servicio | Método | Endpoint | Cuándo |
|---|---|---|---|
| `ConsumoGenericoService` | `consultarGenerico()` | `users` (por `email`) | Al iniciar: guarda `tiendas_asignadas`, `roles_asignados` y `tiendas_id` en `sessionStorage` |
| `ConsumoGenericoService` | `consultarGenerico()` | `module` | Al iniciar: módulos del menú según los roles |
| `ConsumoGenericoService` | `consultarGenerico()` | `metodoGenerico?coleccion=Tienda` | Revisa si las tiendas del usuario están inactivas para mostrar el aviso |
| `SidebarService` (CoreUI) | `toggle()` | — | La directiva cierra el cajón del celular |

---

## Directiva `SidebarInteraccionDirective`

Todo el comportamiento del menú con el mouse, el teclado y el dedo vive en esta directiva (`directives/sidebar-interaccion.directive.ts`), no en el layout. Se pone en el `<c-sidebar>` con el atributo `appSidebarInteraccion`; el layout solo arma el menú y hace la búsqueda.

| Elemento | Nombre | Descripción |
|---|---|---|
| `@HostBinding` | `class.abierta` | Pone la clase `abierta` que usa el SCSS para abrir la barra (en vez de `:hover`) |
| `@HostBinding` | `class.cerrando` | Clase `cerrando` mientras la barra se encoge: el SCSS deja quietos los submenús y desvanece los resultados de búsqueda |
| `@Output` | `limpiarBusqueda` | Avisa al layout que borre lo escrito en el buscador (Escape, Ctrl + B para cerrar o al terminar de cerrarse) |
| `@ContentChild` | `menuSearchInput` | El campo del buscador, para ponerle el cursor con Ctrl + B y las flechas |
| Propiedad | `atajoTexto` / `atajoAria` | Texto del atajo según el sistema: `Ctrl B` o `⌘ B` |
| Propiedad | `mostrarMenuCompleto` | `true` con la barra cerrada en escritorio: el layout pinta el menú completo en vez de los resultados |

- Las clases `abierta` y `cerrando` se ponen **también directo en el elemento** (`classList.toggle`), además del `@HostBinding`. Si una página tiene un error en su plantilla (pasaba en Inversiones por un formulario), Angular deja de redibujar y la barra se quedaba abierta aunque el menú ya hubiera decidido cerrarla.

### Abrir y cerrar con el mouse

| Constante | Valor | Para qué |
|---|---|---|
| `ESPERA_ABRIR_MS` | 150 ms | La barra solo se abre si el mouse se queda encima este tiempo. Una pasada rápida no la mueve |
| `MARGEN_CERRAR_MS` | 120 ms | Margen de perdón: si el mouse sale y vuelve antes, la barra sigue abierta |
| `DURACION_ENCOGER_MS` | 450 ms | Si ya empezó a cerrarse, termina de cerrarse y después se vuelve a abrir. Debe ser igual a `$sidebar-duracion` de `styles/_variables.scss` |

- Al **presionar el mouse** sobre la barra cerrada se abre al instante, sin la espera: el usuario ya decidió usar el menú.
- Sacar el mouse **no** cierra la barra si el usuario está escribiendo en el buscador, recorriendo el menú con las flechas o arrastrando la barrita del scroll.
- Los submenús abiertos se cierran cuando la barra termina de encogerse, con el mismo clic de CoreUI. Así su estado interno queda al día y el siguiente clic siempre abre. Ver [Plegado de los grupos al cerrar](#plegado-de-los-grupos-al-cerrar).

### Cerrar al elegir una opción

Antes la barra solo se cerraba al sacar el mouse y parecía que unas opciones la cerraban y otras no. Ahora, al elegir una opción final (con el mouse o con Enter, desde el menú o desde los resultados del buscador), la barra se cierra sola aunque el mouse siga encima. Abrir un grupo no cuenta.

| Constante | Valor | Para qué |
|---|---|---|
| `ESPERA_PAGINA_MS` | 400 ms | Si hubo un cambio de página hace menos de esto, la barra espera antes de encogerse |
| `ESPERA_LIBRE_MS` | 200 ms | Después de esa espera, aguarda a que el navegador quede libre (`requestIdleCallback`), como mucho este tiempo más |

**Flujo:**
1. `click` en una opción final (`.nav-link` que no es `.nav-group-toggle`): se quita el foco del menú, se marca `cierrePorEleccion` y se llama a `collapseSidebar()`.
2. Pasa el margen de 120 ms y `esperarPaginaYCerrar()` revisa cuándo empezó la última navegación (`NavigationStart`).
3. Si la página nueva acaba de empezar a cargar, espera a que se cumplan los 400 ms y a que el navegador quede libre. La página nueva hace su trabajo pesado (pedir datos, pintar tablas) justo después de elegir y, si la barra se encoge en ese momento, la animación da un tirón (se notaba en Pedidos).
4. `empezarCierre()`: quita `abierta`, pone `cerrando` y la barra se encoge.

- Va en el **clic** y no al cambiar de página: si es la página actual o la app no deja abrirla, no hay cambio de página y la barra igual se cierra.
- Mientras `cierrePorEleccion` está activo, **pasar el mouse no la reabre**: mientras la página carga, el mouse suele salir y volver a entrar a la barra, y eso cancelaba el cierre. Un clic con la barra ya encogiéndose, las flechas o Ctrl + B sí la abren.
- Un **segundo clic** (o doble clic) mientras la barra espera para cerrarse no cancela el cierre. El menú se acomoda con el primer clic y el segundo caía en un espacio vacío, en un grupo o en la barrita del scroll, y la barra no se cerraba hasta sacar el mouse.
- Con **Ctrl, Cmd o Shift** la opción se abre en otra pestaña o ventana: la barra no se cierra, porque el usuario sigue en la misma página y puede querer abrir más.
- Elegida con **Enter**, el foco sale del menú. Antes quedaba en la opción y las flechas de la página volvían a abrir la barra.
- Si la opción se elige con la barra ya cerrada (Enter sobre una opción que quedó con el foco), no se marca `cierrePorEleccion`: no habría un cierre que la apagara y pasar el mouse ya no abría la barra.

### Plegado de los grupos al cerrar

Con la barra ya encogida, los grupos abiertos se pliegan mientras su contenido se desvanece, así los iconos de abajo suben deslizándose. Antes saltaban de golpe, hasta 500 px.

| Constante | Valor | Para qué |
|---|---|---|
| `PLEGAR_MS` | 220 ms | Lo que tarda en plegarse el grupo abierto |
| `CURVA_PLEGAR` | `cubic-bezier(0.33, 1, 0.68, 1)` | Arranca rápido y frena suave al final (ease-out cúbica): los iconos suben de un tirón y se asientan sin brusquedad |

**Flujo:**
1. `plegarGrupos()`: a cada `.nav-group.show > .nav-group-items` le pone en línea su alto actual, `overflow: hidden` y la transición, y luego lo pasa a alto 0 y opacidad 0. Va con `!important` porque las reglas de CoreUI para la barra cerrada fijan el alto en 0 con `!important`.
2. `terminarCierre()`: cierra los grupos de verdad (`collapseAllMenus()`, estado de CoreUI), quita los estilos en línea, quita `cerrando` y borra la búsqueda si había algo escrito.

- Con **"reducir movimiento"** activado en el sistema, o si no hay grupos abiertos, se salta el plegado y se cierra sin animación.
- Si la barra se vuelve a abrir antes de terminar de encogerse, se cancela todo: los grupos quedan como estaban.

### Arrastre de la barrita del scroll

Antes, al arrastrar la barrita del scroll del menú y sacar el mouse de la barra (algo normal al arrastrar), el menú se cerraba en pleno arrastre.

- `ngx-scrollbar` marca el arrastre con el atributo `dragging` en el `<ng-scrollbar>` (`"x"` o `"y"`, `"none"` al soltar). No sirve escuchar `mousedown`/`mouseup` porque la librería cancela esos eventos al arrastrar.
- Un `MutationObserver` vigila ese atributo. Mientras se arrastra, salir con el mouse no cierra la barra.
- Al soltar la barrita **fuera** del menú, se cierra como si el mouse acabara de salir. Si se suelta **encima**, sigue abierta.

### Clic que no se pierde

El navegador solo cuenta un clic si se presiona y se suelta sobre el mismo elemento. Si el menú se mueve mientras el botón está presionado (baja el buscador o se cierra otro grupo), el clic se perdía.

**Flujo:**
1. `mousedown`: se anota la opción presionada y el punto donde se presionó.
2. `click`: si el clic del navegador sí llegó a esa opción, se marca como recibido.
3. `mouseup`: si el clic no llegó y el mouse se movió 10 px o menos (`MOVIMIENTO_MAX_PX`), se completa el clic sobre la opción anotada. Si se movió más, fue el usuario que se arrepintió y no se hace nada.

### Atajo Ctrl + B (Cmd + B en Mac)

**Flujo:**
1. Solo en escritorio. En celular el atajo no hace nada.
2. No se activa escribiendo en un editor con formato, porque ahí Ctrl + B es negrita.
3. Se evita la acción del navegador para esa combinación (en Firefox abre el panel de marcadores).
4. Si la barra ya está abierta con el foco adentro, el atajo la cierra (igual que Escape).
5. Si no, abre la barra al instante y pone el cursor en el buscador con el texto seleccionado.

- **Escape** en el buscador borra lo escrito, cierra la barra y devuelve el foco a donde estaba antes del atajo.
- Abierta con el teclado, la barra se cierra cuando el foco sale de ella, salvo que el mouse siga encima.

### Recorrer el menú con el teclado

Con el menú abierto (con el mouse encima o con Ctrl + B) se puede usar sin el mouse, empezando desde el buscador.

| Tecla | Dónde | Qué hace |
|---|---|---|
| ↓ | Buscador | Pasa a la primera opción |
| ↓ / ↑ | Opción | Pasa a la opción siguiente / anterior. Las de los submenús cerrados y las deshabilitadas no cuentan |
| ↑ | Primera opción | Vuelve al buscador |
| ↓ / ↑ | Página (foco fuera del menú) | Con la barra abierta, ↓ va a la primera opción y ↑ al buscador, sin tener que hacer clic. No aplica si el usuario está escribiendo en un campo de la página (input, textarea, select o editor con formato) |
| Enter | Grupo | Abre el submenú; con ↓ se entra a sus opciones |
| Enter | Opción final | Va a la página, el foco sale del menú y la barra se cierra (ver [Cerrar al elegir una opción](#cerrar-al-elegir-una-opcion)) |
| Escape | Grupo abierto | Lo cierra |
| Escape | Opción de un submenú | Cierra ese submenú y deja el foco en su grupo |
| Escape | Opción del primer nivel | Cierra la barra y limpia la búsqueda, igual que en el buscador |

- Si hay algo escrito en el buscador, las flechas recorren solo los resultados.
- **Enter no lo maneja la directiva:** las opciones de CoreUI son enlaces (`<a href>`) y el navegador ya hace el clic, que abre el submenú o navega con el `routerLink`. Ese clic es el que cierra la barra (`alClicMenu()`).
- La opción con el foco lleva un borde morado por dentro (`:focus-visible` en `styles/_menu.scss`); en la opción activa el borde es blanco. Va por dentro (`outline-offset: -2px`) porque el enlace y la tarjeta del grupo tienen `overflow: hidden`.
- El scroll del menú se mueve solo lo necesario para que se vea la opción (`scrollIntoView({ block: 'nearest' })`).
- **Mover el mouse deja de usar el teclado:** se quita la marca de la opción (si no, se verían dos marcadas, la del mouse y la del teclado) y, si el mouse está fuera del menú, la barra se cierra (salvo con el cursor en el buscador). Se ignoran los movimientos de 0 px que el navegador simula al hacer scroll.
- El atajo y las flechas se escuchan con **un solo** `@HostListener('document:keydown')` (`alPresionarTecla()`), que llama a `atajoBuscarMenu()` y a `alPresionarFlecha()`. Con dos `@HostListener` del mismo evento en la misma directiva, Angular deja solo uno y el otro deja de funcionar.

### Barra cerrada con búsqueda

Los resultados del buscador no tienen iconos (van bajo un título y con sangría). Con la barra cerrada solo se ven los iconos, así que con algo escrito se veían solo unas rayitas.

- Mientras la barra se encoge (clase `cerrando`), los resultados se desvanecen y suben un poco.
- Al **terminar** de encogerse (`terminarCierre()`) se borra la búsqueda y el layout pinta el **menú completo**; sus iconos entran con un fundido. Al volver a abrir la barra el menú está completo.
- La búsqueda no se borra al elegir un resultado: con la barra abierta los resultados se cambiaban de golpe por el menú completo y se veía un momento en blanco.
- El cambio de menú se hace cuando la barra **termina** de encogerse (`barraCompacta`), no al empezar, para que el menú no se reemplace en plena animación.
- En celular el cajón siempre sale completo y ahí se muestran los resultados. La búsqueda se borra al elegir una opción, mientras el cajón se va deslizando.

### Celular

| Constante | Valor | Para qué |
|---|---|---|
| `CORTE_MOVIL` | `(max-width: 767.98px)` | Corte de celular, el mismo de `$corte-movil` del SCSS. Ahí no aplica nada del mouse: el toque del dedo contaba como "mouse que entra" y obligaba a tocar dos veces |
| `DESLIZAR_MIN_PX` | 60 px | Deslizar a la izquierda más que esto (y mucho más en horizontal que en vertical) cierra el cajón. Hacer scroll en el menú no lo cierra |

- Al navegar a otra pantalla el cajón se cierra solo.
- El cajón mide `min(300px, 85vw)` y sale completo de una vez (`styles/_movil.scss`).

---

## Ajuste a CoreUI (`coreui-sidebar-ajuste.ts`)

Se importa una sola vez desde `default-layout.component.ts` y cambia dos métodos del componente de los grupos de CoreUI (`c-sidebar-nav-group`). CoreUI no exporta ese componente: se toma de los componentes que usa `SidebarNavComponent`.

| Método | Qué hacía CoreUI | Qué hace ahora |
|---|---|---|
| `samePath()` | Con `dropdownMode="close"`, al cambiar de página dejaba abierto el grupo cuya dirección coincidía con la de la página y cerraba los demás. Pero los grupos mezclan direcciones (Product HUB tiene `/app/tienda/portafolio` y `/app/dropshipping/productos`), así que cerraba el grupo donde se acababa de elegir una opción, con la barra todavía abierta | Un grupo corresponde a la página si alguna de sus opciones finales (o las de sus subgrupos) es esa página. Los subgrupos no cuentan: traen la dirección de toda su sección (Pedidos Masivos tiene `/app/tienda`) y marcarían el grupo para páginas de otros grupos |
| `openGroup()` | CoreUI pone `routerLinkActive="show"` en cada grupo. El grupo no tiene enlace propio, así que al terminar cada navegación Angular le quitaba la clase `show` aunque siguiera abierto: se veía cerrarse, la directiva no lo encontraba para cerrarlo con la barra y al volver a abrirla aparecía abierto otra vez | Después de que CoreUI decide abrir o cerrar el grupo, se deja la clase `show` igual a su estado. Se corrige dos microtareas después, antes de que el navegador dibuje (con `setTimeout` se alcanzaba a ver un parpadeo) |

- Si una actualización de CoreUI cambia el componente o quita alguno de los dos métodos, el ajuste no se aplica: vuelve el comportamiento original de CoreUI sin romper nada y se avisa en la consola (`Ajuste del menú no aplicado (coreui-sidebar-ajuste.ts)…`).

---

## Buscador del menú

Está en el layout (`onSearch()` y `searchNavItems()`).

- Sin texto muestra el menú completo.
- Con texto devuelve una **lista plana** de las opciones que coinciden por nombre, para no depender de abrir y cerrar grupos mientras se busca. Si coincide un grupo, entran todos sus submenús.
- La comparación ignora tildes y mayúsculas (`normalizeText()`).
- Los resultados van **agrupados bajo un título con la ruta** de donde vienen, por ejemplo "Mi Tienda › Pedidos Masivos" (clase `nav-title-busqueda`). Debajo van sus opciones sin icono y con sangría (clase `nav-item-busqueda`). Las opciones sueltas del primer nivel conservan su icono.
- Si no hay coincidencias se muestra *Sin resultados para "…"*.
- La etiqueta `Ctrl B` a la derecha del campo recuerda el atajo; se oculta al escribir y en pantallas táctiles.
- Al escribir aparece en su lugar un botón **X** (`.menu-search-limpiar`) que borra lo escrito y deja el cursor en el campo. Su `mousedown` no hace nada (`preventDefault()`) para que el campo no pierda el foco. Reemplaza la X que pone el navegador en los campos de búsqueda, que se oculta: así es igual en todos los navegadores y del mismo estilo del menú.

---

## Diseño y animaciones

| Qué | Cómo se ve |
|---|---|
| Barra en escritorio | Cerrada mide 80 px (solo iconos) y abierta 300 px. Se abre en 0.45 s |
| Apertura por capas | Primero se ensancha mientras el logo se despliega con una máscara (cerrada solo se ve el icono), después entran el buscador y los textos en cascada y al final las flechas de los grupos |
| Submenús al abrir | Sus opciones entran una tras otra con un fundido y bajando un poco. En escritorio la cascada se repite cada vez que se abre la barra, no solo la primera vez que se abre el grupo |
| Cierre | Los submenús abiertos se quedan quietos, cortados por el borde, mientras sus opciones y los resultados de búsqueda se desvanecen y suben un poco. Al terminar, los grupos se pliegan en 0.22 s |
| Sección actual | Barrita azul a la izquierda del principal abierto o que contiene la página actual; crece con un pequeño rebote |
| Opción activa | En morado, con un brillo suave |
| Opción con el foco del teclado | Borde morado por dentro y fondo lila suave (blanco sobre la activa) |
| Subgrupos | Pastilla, puntito y línea tipo árbol |
| Barra cerrada | Rayitas separadoras entre principales. El submenú del grupo de la página actual queda escondido aunque el mouse esté encima: CoreUI lo mostraba con `:hover` y se veían las opciones amontonadas en los 80 px |
| Iconos | Entran con un fundido cuando se crean (al cambiar de los resultados al menú completo) |
| Botón X del buscador | Gris; al pasar el mouse o con el foco del teclado, morado con fondo lila suave |
| Traductor | `translate="no"` en el sidebar: el traductor del navegador no cambia los nombres del menú |
| Menos movimiento | Se respeta la opción "reducir movimiento" del sistema (`prefers-reduced-motion`) |

Los valores que se repiten están en `styles/_variables.scss`:

| Variable | Valor | Uso |
|---|---|---|
| `$sidebar-duracion` | `0.45s` | Tiempo de abrir y cerrar. Debe coincidir con `DURACION_ENCOGER_MS` de la directiva |
| `$curva-abrir` | `cubic-bezier(0.22, 1, 0.36, 1)` | Arranca con impulso y frena suave |
| `$curva-cerrar` | `cubic-bezier(0.4, 0, 0.2, 1)` | Movimiento parejo para cerrar |
| `$curva-rebote` | `cubic-bezier(0.34, 1.56, 0.64, 1)` | Barrita azul, iconos y flechas |
| `$corte-movil` / `$corte-escritorio` | `767.98px` / `768px` | Corte entre celular y escritorio. Debe coincidir con `CORTE_MOVIL` de la directiva |

Los `@keyframes` del menú (`sidebar-aparecer`, `nav-sub-in`, `nav-glow`, `nav-text-in`) están en `src/scss/_animaciones-menu.scss`, en los estilos globales. Ver [Observaciones](#observaciones).

---

## Flujo principal

```
Usuario entra a una ruta bajo /app
  → DefaultLayoutComponent
    → consulta users       → guarda tiendas y roles en sessionStorage
    → consulta module      → buildNavItems() según roles → navItems / visibleNavItems
    → validateModuleAccess() → si no tiene acceso a la ruta, va a login
    → cierra la barra (directiva)

Escritorio
  Mouse encima 150 ms   → clase "abierta" → apertura por capas
  Mouse sale            → espera 120 ms → "cerrando": se encoge en 0.45 s → pliega los grupos en 0.22 s
                          → cierra submenús y borra la búsqueda → menú completo (iconos)
  Elegir una opción     → foco fuera del menú → espera 120 ms (y a que cargue la página nueva) → se cierra igual
                          aunque el mouse siga encima (Ctrl / Cmd / Shift + clic no la cierra)
  Arrastrar el scroll   → no se cierra aunque el mouse salga; al soltar afuera se cierra
  Ctrl + B              → abre al instante → cursor en el buscador
  Escribir              → onSearch() → resultados agrupados por sección (X para borrar)
  ↓ / ↑                 → buscador ⇄ opciones visibles (también desde la página con la barra abierta)
  Enter                 → abre el submenú o va a la página (y cierra la barra)
  Escape                → cierra el grupo / el submenú / la barra
  Mover el mouse        → quita la marca del teclado (y cierra si el mouse está afuera)

Celular
  ☰ del header          → cajón completo
  Elegir opción         → navega, cierra el cajón y borra la búsqueda
  Deslizar a la izquierda → cierra el cajón
```

---

## Pruebas

`directives/sidebar-interaccion.directive.spec.ts` tiene **33 pruebas**. Usa un componente mínimo en vez del layout completo, así no se llama a ningún servicio de datos; el menú de prueba tiene dos opciones y un grupo con una opción hija.

- **Mouse:** pasar rápido no abre la barra; dejarlo encima la abre a los 150 ms; margen de perdón de 120 ms; termina de cerrar antes de volver a abrir; presionar con la barra cerrada la abre al instante.
- **Clic que no se pierde:** se completa si no llegó, no se repite si sí llegó y no se completa si el mouse se arrastró más de 10 px.
- **Ctrl + B y Escape:** abre y enfoca el buscador; con el foco adentro la cierra; no hace nada en un editor con formato; Escape cierra, limpia y devuelve el foco.
- **Teclado:** ↓ pasa del buscador a las opciones y salta las de un submenú cerrado; ↑ vuelve y desde la primera regresa al buscador; con el submenú abierto ↓ entra a sus opciones; Escape en una opción del submenú, en un grupo abierto y en el primer nivel; con el menú abierto por el mouse las flechas funcionan sin clic; con el menú cerrado o escribiendo en un campo no se tocan; recorriendo con las flechas, salir con el mouse no cierra.
- **Mover el mouse:** sobre el menú quita la marca y deja la barra abierta; fuera del menú la quita y cierra; el movimiento de 0 px del scroll no la quita.
- **Barra cerrada con búsqueda:** pide el menú completo cerrada y los resultados abierta; el cambio pasa al terminar de encogerse; en celular siempre resultados.
- **Barrita del scroll:** arrastrando, salir no cierra y al soltar afuera sí; al soltar con el mouse encima sigue abierta.
- **Celular:** el toque no abre la barra ni Ctrl + B hace nada; deslizar a la izquierda cierra el cajón; scroll o toque corto no lo cierran.

El cierre al elegir una opción, el plegado de los grupos y el ajuste a CoreUI todavía no tienen pruebas.

```bash
npx ng test --include=src/app/layout/default-layout/directives/sidebar-interaccion.directive.spec.ts --watch=false --browsers=ChromeHeadless
```

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-10-09 | Yadilton Torres | Rediseño del menú lateral: apertura sin parpadeo con la clase `abierta`, clic que no se pierde, barrita azul y subgrupos, buscador del menú con Ctrl + B y resultados agrupados, cajón completo en celular. El comportamiento pasó a la directiva `SidebarInteraccionDirective` y los estilos a una carpeta por tema. |
| 2026-10-15 | Yadilton Torres | Uso del menú con el teclado (flechas, Enter y Escape, marca de la opción con el foco, el mouse quita la marca). Arrastrar la barrita del scroll ya no cierra el menú. Con la barra cerrada se ven los iconos del menú completo aunque haya búsqueda. La directiva pasó a `directives/` y los estilos de `estilos/` a `styles/`. 33 pruebas. |
| 2026-10-28 | Yadilton Torres | La barra se cierra sola al elegir una opción y espera a que cargue la página nueva; los grupos se pliegan deslizándose al terminar de cerrar; la búsqueda se borra al cerrarse; botón X en el buscador; ajuste a CoreUI (`coreui-sidebar-ajuste.ts`) para el grupo de la página actual; `@keyframes` del menú a los estilos globales. Correcciones: el foco quedaba en el menú tras elegir con Enter, un segundo clic cancelaba el cierre y Ctrl + clic cerraba la barra. |

---

## Observaciones

- **Dos valores deben ir sincronizados** entre el TS y el SCSS: la duración (`DURACION_ENCOGER_MS` = `$sidebar-duracion`) y el corte de celular (`CORTE_MOVIL` = `$corte-movil`). Si se cambia uno, cambiar el otro.
- **El corte de celular no es el mismo de CoreUI.** El proyecto usa el `$mobile-breakpoint` de CoreUI por defecto (`lg`, 992 px), mientras que la directiva y el SCSS del layout cortan en 768 px. Por eso las reglas que neutralizan el `:hover` de CoreUI en `styles/_barra.scss` van en `@media (min-width: 992px)`. Entre 768 y 991 px de ancho conviene revisar el menú si algo se ve raro.
- **Un solo `@HostListener('document:keydown')`** en la directiva. Si se agrega otra tecla global, llamarla desde `alPresionarTecla()` en vez de poner otro `@HostListener` del mismo evento.
- **Los `@keyframes` del menú van en los estilos globales** (`src/scss/_animaciones-menu.scss`, importado en `styles.scss`), no en el componente. Angular le cambia el nombre a los `@keyframes` de un componente pero no corrige las reglas que están dentro de un `@media`, que seguían buscando el nombre original y no se animaban (submenús en cascada, textos al abrir, flechas). Si se agrega una animación nueva al menú, poner su `@keyframes` ahí.
- **`coreui-sidebar-ajuste.ts` depende de partes internas de CoreUI** (`ɵcmp`, `samePath`, `openGroup`). Al actualizar CoreUI, revisar que no aparezca el aviso en la consola.
- El orden de los `@use` en `default-layout.component.scss` se respeta al armar el CSS. Hoy ninguna regla de un archivo pisa a otra de otro archivo; si se agrega una que lo haga, tenerlo en cuenta.
- La apertura no usa `:hover` de CoreUI (barra `unfoldable`): en escritorio esas reglas quedan neutralizadas mientras la barra no tenga la clase `abierta`.
- `sidebar-nav.scss` está vacío y no se usa.
