---
autor: Yadilton Torres
fecha_creacion: 2026-10-09
ultima_actualizacion: 2026-10-09
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
- **Clics que no se pierden:** si el menú se mueve mientras se hace clic, la opción igual se abre.
- **Ubicación visible:** barrita azul en la sección abierta o actual, opción activa en morado y subgrupos con pastilla y línea de árbol.
- **Buscador del menú** con atajo **Ctrl + B** (**Cmd + B** en Mac) y resultados agrupados por sección.
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
├── default-layout.component.scss       → Solo importa los archivos de estilos/ (10 líneas)
├── sidebar-interaccion.directive.ts    → Mouse, clics, Ctrl + B y cajón del celular
├── sidebar-interaccion.directive.spec.ts → 15 pruebas del comportamiento del menú
├── estilos/
│   ├── _variables.scss                 → Tiempos, curvas de animación y corte de celular
│   ├── _menu.scss                      → Colores, opciones, iconos, barrita azul, activo, grupos y subgrupos
│   ├── _barra.scss                     → Ancho cerrada/abierta y apertura por capas (logo, buscador, textos, flechas)
│   ├── _buscador.scss                  → Campo, etiqueta Ctrl B, "Sin resultados" y resultados agrupados
│   ├── _movil.scss                     → Cajón del celular
│   ├── _animaciones.scss               → Fundidos, cascadas, brillo del activo y "reducir movimiento"
│   └── _pagina.scss                    → Contenedor, cuerpo y botón de WhatsApp (ya existía)
└── default-header/                     → Barra superior (ver su página)
```

---

## Decoradores y propiedades clave

| Elemento | Nombre | Tipo | Descripción |
|---|---|---|---|
| `@ViewChild` | `sidebarInteraccion` | `SidebarInteraccionDirective` | Directiva del sidebar. El layout la usa para cerrar la barra cuando terminan de cargar los módulos |
| Propiedad | `navItems` | `INavData[]` | Menú completo, armado desde la colección `module` según los roles |
| Propiedad | `visibleNavItems` | `INavData[]` | Lo que se pinta: igual a `navItems`, o los resultados si hay búsqueda |
| Propiedad | `searchTerm` | `string` | Texto escrito en el buscador del menú |

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

Todo el comportamiento del menú con el mouse, el teclado y el dedo vive en esta directiva, no en el layout. Se pone en el `<c-sidebar>` con el atributo `appSidebarInteraccion`; el layout solo arma el menú y hace la búsqueda.

| Elemento | Nombre | Descripción |
|---|---|---|
| `@HostBinding` | `class.abierta` | Pone la clase `abierta` que usa el SCSS para abrir la barra (en vez de `:hover`) |
| `@Output` | `limpiarBusqueda` | Avisa al layout que borre lo escrito en el buscador (Escape o Ctrl + B para cerrar) |
| `@ContentChild` | `menuSearchInput` | El campo del buscador, para ponerle el cursor con Ctrl + B |
| Propiedad | `atajoTexto` / `atajoAria` | Texto del atajo según el sistema: `Ctrl B` o `⌘ B` |

### Abrir y cerrar con el mouse

| Constante | Valor | Para qué |
|---|---|---|
| `ESPERA_ABRIR_MS` | 150 ms | La barra solo se abre si el mouse se queda encima este tiempo. Una pasada rápida no la mueve |
| `MARGEN_CERRAR_MS` | 120 ms | Margen de perdón: si el mouse sale y vuelve antes, la barra sigue abierta |
| `DURACION_ENCOGER_MS` | 450 ms | Si ya empezó a cerrarse, termina de cerrarse y después se vuelve a abrir. Debe ser igual a `$sidebar-duracion` de `estilos/_variables.scss` |

- Al **presionar el mouse** sobre la barra cerrada se abre al instante, sin la espera: el usuario ya decidió usar el menú.
- Si el usuario está escribiendo en el buscador, sacar el mouse **no** cierra la barra.
- Los submenús abiertos se cierran cuando la barra termina de encogerse, con el mismo clic de CoreUI. Así su estado interno queda al día y el siguiente clic siempre abre (se corrigió el "primer clic gastado").

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

### Celular

| Constante | Valor | Para qué |
|---|---|---|
| `CORTE_MOVIL` | `(max-width: 767.98px)` | Corte de celular, el mismo de CoreUI y de `$corte-movil` del SCSS. Ahí no aplica nada del mouse: el toque del dedo contaba como "mouse que entra" y obligaba a tocar dos veces |
| `DESLIZAR_MIN_PX` | 60 px | Deslizar a la izquierda más que esto (y mucho más en horizontal que en vertical) cierra el cajón. Hacer scroll en el menú no lo cierra |

- Al navegar a otra pantalla el cajón se cierra solo.
- El cajón mide `min(300px, 85vw)` y sale completo de una vez (`estilos/_movil.scss`).

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
| Subgrupos | Pastilla, puntito y línea tipo árbol |
| Barra cerrada | Rayitas separadoras entre principales |
| Traductor | `translate="no"` en el sidebar: el traductor del navegador no cambia los nombres del menú |
| Menos movimiento | Se respeta la opción "reducir movimiento" del sistema (`prefers-reduced-motion`) |

Los valores que se repiten están en `estilos/_variables.scss`:

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
  Mouse sale            → espera 120 ms → se encoge en 0.45 s → cierra submenús
  Ctrl + B              → abre al instante → cursor en el buscador
  Escribir              → onSearch() → resultados agrupados por sección
  Escape                → borra, cierra y devuelve el foco

Celular
  ☰ del header          → cajón completo
  Elegir opción         → navega y cierra el cajón
  Deslizar a la izquierda → cierra el cajón
```

---

## Pruebas

`sidebar-interaccion.directive.spec.ts` tiene **15 pruebas**:

- Pasar el mouse rápido no abre la barra; dejarlo encima la abre a los 150 ms.
- Margen de perdón de 120 ms y "termina de cerrar antes de volver a abrir".
- Presionar el mouse con la barra cerrada la abre al instante.
- Clic que no se pierde: se completa si no llegó, no se repite si sí llegó y no se completa si el mouse se arrastró más de 10 px.
- Ctrl + B abre y enfoca el buscador; con el foco adentro la cierra; no hace nada en un editor con formato.
- Escape cierra, limpia y devuelve el foco.
- Celular: el toque no abre la barra ni Ctrl + B hace nada; deslizar a la izquierda cierra el cajón; scroll o toque corto no lo cierran.

```bash
npx ng test --include=src/app/layout/default-layout/sidebar-interaccion.directive.spec.ts --watch=false --browsers=ChromeHeadless
```

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-10-09 | Yadilton Torres | Rediseño del menú lateral: apertura sin parpadeo con la clase `abierta`, clic que no se pierde, barrita azul y subgrupos, buscador del menú con Ctrl + B y resultados agrupados, cajón completo en celular. El comportamiento pasó a la directiva `SidebarInteraccionDirective` y los estilos a la carpeta `estilos/` (7 archivos por tema). |

---

## Observaciones

- **Dos valores deben ir sincronizados** entre el TS y el SCSS: la duración (`DURACION_ENCOGER_MS` = `$sidebar-duracion`) y el corte de celular (`CORTE_MOVIL` = `$corte-movil`). Si se cambia uno, cambiar el otro.
- El orden de los `@use` en `default-layout.component.scss` se respeta al armar el CSS. Hoy ninguna regla de un archivo pisa a otra de otro archivo; si se agrega una que lo haga, tenerlo en cuenta.
- La apertura no usa `:hover` de CoreUI (barra `unfoldable`): en escritorio esas reglas quedan neutralizadas mientras la barra no tenga la clase `abierta`.
- `sidebar-nav.scss` está vacío y no se usa.
