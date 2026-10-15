---
autor: Yadilton Torres
fecha_creacion: 2026-10-15
ultima_actualizacion: 2026-10-15
estado: desarrollo
nivel: 2
---

# Menú lateral (Default Layout)

**Autor:** Yadilton Torres

**Selector:** `app-dashboard`

**Ubicación:** `SitioLogiGho/src/app/layout/default-layout`

---

## ¿Qué hace?

Es la estructura que envuelve todas las pantallas internas de LogiGho: el **menú lateral** (sidebar), la barra superior (header), el contenido de la página (`router-outlet`), el botón de WhatsApp, la barra flotante y el footer.

El menú lateral se arma según los roles del usuario (colección `module`) y tiene:

- **Apertura sin parpadeo:** se abre solo si el mouse se queda encima un momento y tiene un margen antes de cerrarse.
- **Clics que no se pierden:** si el menú se mueve mientras se hace clic, la opción igual se abre.
- **Ubicación visible:** barrita azul en la sección abierta o actual, opción activa en morado y subgrupos con pastilla y línea de árbol.
- **Buscador del menú** con atajo **Ctrl + B** (**Cmd + B** en Mac) y resultados agrupados por sección.
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
├── default-layout.component.html       → Sidebar, buscador, header, contenido y footer
├── default-layout.component.scss       → Solo importa los archivos de styles/ (10 líneas)
├── directives/
│   ├── sidebar-interaccion.directive.ts      → Mouse, clics, Ctrl + B, flechas, scroll y cajón del celular
│   └── sidebar-interaccion.directive.spec.ts → 33 pruebas del comportamiento del menú
├── styles/
│   ├── _variables.scss                 → Tiempos, curvas de animación y corte de celular
│   ├── _menu.scss                      → Colores, opciones, iconos, barrita azul, activo, grupos, subgrupos y marca del teclado
│   ├── _barra.scss                     → Ancho cerrada/abierta y apertura por capas (logo, buscador, textos, flechas)
│   ├── _buscador.scss                  → Campo, etiqueta Ctrl B, "Sin resultados" y resultados agrupados
│   ├── _movil.scss                     → Cajón del celular
│   ├── _animaciones.scss               → Fundidos, cascadas, brillo del activo y "reducir movimiento"
│   └── _pagina.scss                    → Contenedor, cuerpo y botón de WhatsApp (ya existía)
└── default-header/                     → Barra superior
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
| `@Output` | `limpiarBusqueda` | Avisa al layout que borre lo escrito en el buscador (Escape o Ctrl + B para cerrar) |
| `@ContentChild` | `menuSearchInput` | El campo del buscador, para ponerle el cursor con Ctrl + B y las flechas |
| Propiedad | `atajoTexto` / `atajoAria` | Texto del atajo según el sistema: `Ctrl B` o `⌘ B` |
| Propiedad | `mostrarMenuCompleto` | `true` con la barra cerrada en escritorio: el layout pinta el menú completo en vez de los resultados |

### Abrir y cerrar con el mouse

| Constante | Valor | Para qué |
|---|---|---|
| `ESPERA_ABRIR_MS` | 150 ms | La barra solo se abre si el mouse se queda encima este tiempo. Una pasada rápida no la mueve |
| `MARGEN_CERRAR_MS` | 120 ms | Margen de perdón: si el mouse sale y vuelve antes, la barra sigue abierta |
| `DURACION_ENCOGER_MS` | 450 ms | Si ya empezó a cerrarse, termina de cerrarse y después se vuelve a abrir. Debe ser igual a `$sidebar-duracion` de `styles/_variables.scss` |

- Al **presionar el mouse** sobre la barra cerrada se abre al instante, sin la espera: el usuario ya decidió usar el menú.
- Sacar el mouse **no** cierra la barra si el usuario está escribiendo en el buscador, recorriendo el menú con las flechas o arrastrando la barrita del scroll.
- Los submenús abiertos se cierran cuando la barra termina de encogerse, con el mismo clic de CoreUI. Así su estado interno queda al día y el siguiente clic siempre abre.

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
| Enter | Opción final | Va a la página y la barra se cierra |
| Escape | Grupo abierto | Lo cierra |
| Escape | Opción de un submenú | Cierra ese submenú y deja el foco en su grupo |
| Escape | Opción del primer nivel | Cierra la barra y limpia la búsqueda, igual que en el buscador |

- Si hay algo escrito en el buscador, las flechas recorren solo los resultados.
- **Enter no lo maneja la directiva:** las opciones de CoreUI son enlaces (`<a href>`) y el navegador ya hace el clic, que abre el submenú o navega con el `routerLink`.
- La opción con el foco lleva un borde morado por dentro (`:focus-visible` en `styles/_menu.scss`); en la opción activa el borde es blanco. Va por dentro (`outline-offset: -2px`) porque el enlace y la tarjeta del grupo tienen `overflow: hidden`.
- El scroll del menú se mueve solo lo necesario para que se vea la opción (`scrollIntoView({ block: 'nearest' })`).
- **Mover el mouse deja de usar el teclado:** se quita la marca de la opción (si no, se verían dos marcadas, la del mouse y la del teclado) y, si el mouse está fuera del menú, la barra se cierra (salvo con el cursor en el buscador). Se ignoran los movimientos de 0 px que el navegador simula al hacer scroll.
- El atajo y las flechas se escuchan con **un solo** `@HostListener('document:keydown')` (`alPresionarTecla()`), que llama a `atajoBuscarMenu()` y a `alPresionarFlecha()`. Con dos `@HostListener` del mismo evento en la misma directiva, Angular deja solo uno y el otro deja de funcionar.

### Barra cerrada con búsqueda

Los resultados del buscador no tienen iconos (van bajo un título y con sangría). Con la barra cerrada solo se ven los iconos, así que con algo escrito se veían solo unas rayitas.

- Con la barra cerrada el layout pinta el **menú completo** aunque haya algo escrito; al abrirla vuelven los resultados.
- El cambio se hace cuando la barra **termina** de encogerse (`barraCompacta`), no al empezar, para que el menú no se reemplace en plena animación.
- En celular el cajón siempre sale completo y ahí se muestran los resultados.

### Celular

| Constante | Valor | Para qué |
|---|---|---|
| `CORTE_MOVIL` | `(max-width: 767.98px)` | Corte de celular, el mismo de CoreUI y de `$corte-movil` del SCSS. Ahí no aplica nada del mouse: el toque del dedo contaba como "mouse que entra" y obligaba a tocar dos veces |
| `DESLIZAR_MIN_PX` | 60 px | Deslizar a la izquierda más que esto (y mucho más en horizontal que en vertical) cierra el cajón. Hacer scroll en el menú no lo cierra |

- Al navegar a otra pantalla el cajón se cierra solo.
- El cajón mide `min(300px, 85vw)` y sale completo de una vez (`styles/_movil.scss`).

---

## Buscador del menú

Está en el layout (`onSearch()` y `searchNavItems()`).

- Sin texto muestra el menú completo.
- Con texto devuelve una **lista plana** de las opciones que coinciden por nombre, para no depender de abrir y cerrar grupos mientras se busca. Si coincide un grupo, entran todos sus submenús.
- La comparación ignora tildes y mayúsculas (`normalizeText()`).
- Los resultados van **agrupados bajo un título con la ruta** de donde vienen, por ejemplo "Mi Tienda › Pedidos Masivos" (clase `nav-title-busqueda`). Debajo van sus opciones sin icono y con sangría (clase `nav-item-busqueda`). Las opciones sueltas del primer nivel conservan su icono.
- Si no hay coincidencias se muestra *Sin resultados para "…"*.
- La etiqueta `Ctrl B` a la derecha del campo recuerda el atajo; se oculta al escribir y en pantallas táctiles.

---

## Diseño y animaciones

| Qué | Cómo se ve |
|---|---|
| Barra en escritorio | Cerrada mide 80 px (solo iconos) y abierta 300 px. Se abre en 0.45 s |
| Apertura por capas | Primero se ensancha mientras el logo se despliega con una máscara (cerrada solo se ve el icono), después entran el buscador y los textos en cascada y al final las flechas de los grupos |
| Sección actual | Barrita azul a la izquierda del principal abierto o que contiene la página actual; crece con un pequeño rebote |
| Opción activa | En morado, con un brillo suave |
| Opción con el foco del teclado | Borde morado por dentro y fondo lila suave (blanco sobre la activa) |
| Subgrupos | Pastilla, puntito y línea tipo árbol |
| Barra cerrada | Rayitas separadoras entre principales |
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
  Mouse sale            → espera 120 ms → se encoge en 0.45 s → cierra submenús → menú completo (iconos)
  Arrastrar el scroll   → no se cierra aunque el mouse salga; al soltar afuera se cierra
  Ctrl + B              → abre al instante → cursor en el buscador
  Escribir              → onSearch() → resultados agrupados por sección
  ↓ / ↑                 → buscador ⇄ opciones visibles (también desde la página con la barra abierta)
  Enter                 → abre el submenú o va a la página (y cierra la barra)
  Escape                → cierra el grupo / el submenú / la barra
  Mover el mouse        → quita la marca del teclado (y cierra si el mouse está afuera)

Celular
  ☰ del header          → cajón completo
  Elegir opción         → navega y cierra el cajón
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

```bash
npx ng test --include=src/app/layout/default-layout/directives/sidebar-interaccion.directive.spec.ts --watch=false --browsers=ChromeHeadless
```

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-10-09 | Yadilton Torres | Rediseño del menú lateral: apertura sin parpadeo con la clase `abierta`, clic que no se pierde, barrita azul y subgrupos, buscador del menú con Ctrl + B y resultados agrupados, cajón completo en celular. El comportamiento pasó a la directiva `SidebarInteraccionDirective` y los estilos a una carpeta por tema. |
| 2026-10-15 | Yadilton Torres | Uso del menú con el teclado (flechas, Enter y Escape, marca de la opción con el foco, el mouse quita la marca). Arrastrar la barrita del scroll ya no cierra el menú. Con la barra cerrada se ven los iconos del menú completo aunque haya búsqueda. La directiva pasó a `directives/` y los estilos de `estilos/` a `styles/`. 33 pruebas. |

---

## Observaciones

- **Dos valores deben ir sincronizados** entre el TS y el SCSS: la duración (`DURACION_ENCOGER_MS` = `$sidebar-duracion`) y el corte de celular (`CORTE_MOVIL` = `$corte-movil`). Si se cambia uno, cambiar el otro.
- **Un solo `@HostListener('document:keydown')`** en la directiva. Si se agrega otra tecla global, llamarla desde `alPresionarTecla()` en vez de poner otro `@HostListener` del mismo evento.
- El orden de los `@use` en `default-layout.component.scss` se respeta al armar el CSS. Hoy ninguna regla de un archivo pisa a otra de otro archivo; si se agrega una que lo haga, tenerlo en cuenta.
- La apertura no usa `:hover` de CoreUI (barra `unfoldable`): en escritorio esas reglas quedan neutralizadas mientras la barra no tenga la clase `abierta`.
- `sidebar-nav.scss` está vacío y no se usa.
