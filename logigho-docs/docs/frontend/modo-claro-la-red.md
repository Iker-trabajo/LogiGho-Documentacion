---

## Autor: Iker Acevedo (decisión) · redactado con Claude

Fecha creacion: 2026-11-23
Estado: desarrollo (verificado en local; pendiente de validar en PreProd)
Tipo: componente

# Modo claro «Papel de ruta» e interruptor de tema

**Selector del interruptor:** `app-lg-interruptor-tema`  
**Ubicación:** `src/app/components/lg-interruptor-tema/` (componente, `TemaServicio` y specs); estilos en `src/scss/lared/_tokens.scss`; pre-pintado en `src/index.html`  
**Acceso:** Autenticado (cabecera y menú del celular) y público (login y registro)

!!! warning "Estado: verificado en local, pendiente de PreProd"
    Cifras y pendientes en [Rediseño «La Red»](tema-la-red.md). Lo que **no** se vio en pantalla: modales de estados de usuario, novedades de wallet y de inversionista, actualización de productos, pasos 2 a 4 de «Accesos», un aviso real del registro, la puerta del login y el interior de los marcos de Power BI (M-380). Marcas: **observado** (archivo citado), **Iker (fecha)**, **hipótesis**.

---

## ¿Qué hace?

Ofrece una versión clara de toda la plataforma, además de la noche aprobada. La persona la elige con un interruptor; se recuerda entre visitas y la página no parpadea al recargar.

- **Por defecto: noche.** No se sigue la preferencia del sistema operativo (`prefers-color-scheme`); es una decisión abierta de Iker ([ADR-RED-002](../arquitectura/adr/rediseno-la-red/ADR-RED-002-noche-por-defecto-y-claro-por-atributo.md)).
- **Noche = sin atributo.** Los píxeles de noche no cambian al añadir el modo claro, salvo el rectángulo del interruptor, que existe en los dos modos.
- La versión noche aprobada por Iker el 2026-11-22 queda congelada en la etiqueta local `rediseno-la-red-noche-v1` (`3a531b16`).

---

## Paleta «Papel de ruta»

Idea del diseño: el registro de noche es un mapa iluminado sobre azul marino; el claro es **el mismo mapa impreso de día**: papel cálido, la tinta es el azul marino, el azul de marca estructura, y la línea cálida rojo-naranja sigue firmando cabecera y paneles. Sombras tintadas de azul marino, no grises.

| Token | Noche | Claro |
|---|---|---|
| `--lg-pagina` | `#0a1f33` | `#f5f3ef` (papel) |
| `--lg-superficie` | `#102f4b` | `#ffffff` |
| `--lg-superficie-3` | `#0b2236` | `#f0ece6` (hundido) |
| `--lg-borde` | `rgba(255,255,255,.10)` | `#e5e0d8` |
| `--lg-borde-control` | `#8295ab` | `#7d838c` |
| `--lg-tinta` / `--lg-texto` | `#ffffff` / `#e6edf7` | `#0a1f33` / `#14283c` |
| `--lg-azul` | `#9fc0ff` | `#1e40af` |
| `--lg-foco` | `#f5a623` | `#c2410c` |
| `--lg-acento` | `#ffc27a` | `#b8400b` |

(**Observado** en `_tokens.scss`: mixins `lg-tokens-noche` y `lg-tokens-claro`.) El shell (cabecera, menú lateral, buscador, campana, menú móvil) usa la familia `--lg-shell-*`: en noche vale la constante `--lg-noche-*`; en claro se redefine (`--lg-shell-fondo: #f5f3ef`, `--lg-shell-superficie: #fff`…). Los tooltips y la ayuda flotante se quedan oscuros en ambos modos.

**Contraste (WCAG 2.2):** 60 de 60 pares de tokens y 27 de 27 del login llegan a su umbral (4,5:1 para texto, 3:1 para bordes y foco), y 56 capturas de modales y tabla se auditaron sin textos bajos (`ENTREGA.md` §4). El diseño original contaba 59 pares; la verificación final, 60 (el conteo creció durante la obra; no hay una lista separada que lo explique). Dos límites que el diseño declara como **hipótesis**: el panel translúcido del login se calculó sobre blanco, y `--lg-texto-off` (`#6b7686`) solo vale para texto deshabilitado. El halo `--lg-anillo` es decorativo: el 3:1 lo da el contorno `--lg-foco`.

---

## Cómo se activa

| Pieza | Detalle |
|---|---|
| Atributo | `<html data-lg-modo="dia">`. Sin el atributo, noche |
| Almacenamiento | `localStorage['lg-modo']` vale `dia` o `noche`. Cualquier otro valor se trata como noche |
| Pre-pintado | Script en línea de `src/index.html` que lee `lg-modo` y pone el atributo **antes del primer pintado**, además del color de `<meta name="theme-color">` (`#0a1f33` en noche, `#f5f3ef` en claro). Verificado: con el modo guardado, `<html>` ya trae el atributo antes de arrancar Angular |
| Cargador | El spinner inicial de `index.html` tiene su versión clara (fondo `#f5f3ef`). Tras el login la página recarga, y la recarga cae en el cargador del modo guardado |
| Servicio | `TemaServicio` (`providedIn: 'root'`): señales `modo` y `esClaro`, métodos `alternar()` y `establecer(modo)`. Sin peticiones. La fuente de verdad inicial es el atributo que ya puso el script; no relee `localStorage` |
| Otras pestañas | Escucha el evento `storage` de **su** clave y alinea el modo; no vuelve a escribir en el almacenamiento |
| Fallo del almacenamiento | Si `localStorage` está bloqueado, el modo vale solo para la sesión (no rompe) |
| No usa | `ColorModeService` de CoreUI ni su clave o atributo |

Cambiar de modo **no hace ninguna petición**: la paridad de peticiones da 0 diferencias en noche y en claro.

### Dónde está el interruptor

| Superficie | Dónde | Variante |
|---|---|---|
| Escritorio | Primer elemento del grupo de acciones de la cabecera, antes del botón «Consultar guía» | `cabecera` |
| Celular (menú móvil) | En la cabecera del menú móvil, entre el logo y el botón de cerrar | `panel` |
| Login y recuperar | Fuera del flujo, abajo a la izquierda (≥ 1024 px, `position: fixed`) o sobre el mapa (< 1024 px) | `flotante` |
| Registro | Igual que el login (misma plantilla) | `flotante` |

Es un `<button type="button" role="switch">` con `aria-checked`, `aria-label="Modo claro"`, zona táctil de 44×44 px y foco visible (**observado** en `lg-interruptor-tema.component.html` y `.scss`). Integración en TypeScript: solo el `import` y el arreglo `imports` de `default-header`, `menu-movil`, `login` y `registro`.

!!! note "Bajo 375 px"
    El diseño preveía ocultar el interruptor de la cabecera por debajo de 375 px. En el SCSS actual de la cabecera **no hay** esa regla, y qué pasa a esos anchos no está verificado.

---

## Regla para arreglar colores fijos sin tocar la noche

Lo que se rompe en claro es lo que lleva un color **fijo**: texto `#fff`, fondos o sombras `rgba(255,255,255,…)`, fondos oscuros en hex, `filter: invert()` sobre iconos, degradados oscuros. Todo lo que ya usa tokens cambia solo. Dos formas de arreglar, **en este orden** (decisión D49, que hace que la noche no pueda cambiar por construcción):

1. **Token cuyo valor de noche sea exactamente el color que había.** Noche queda igual; claro se arregla. Se comprueba el valor en `_tokens.scss`.
2. **Regla que solo existe en claro**, dejando la actual intacta:

```scss
// componente con encapsulación normal
:host-context([data-lg-modo='dia']) .selector { color: var(--lg-texto); }

// componente con ViewEncapsulation.None o SCSS global
:root[data-lg-modo='dia'] .selector { color: var(--lg-texto); }
```

En esa regla se usan tokens, no hex nuevos. Texto blanco sobre un fondo de color propio (botón principal, chip sólido) está bien en los dos modos. No se deshacen las correcciones de móvil ni las clases de movimiento. Esta receta se aplicó revisando ~150 pantallas y ~25 modales a ojo a 390 y 1440 px; hubo 22 archivos corregidos.

---

## Login, recuperar y registro en claro (sin tocar la escena 3D)

- **Dónde:** `src/app/views/pages/registro/styles/_registro-claro.scss` (365 líneas), importado con una sola línea al final de `registro.scss`. Todo cuelga de `:root[data-lg-modo='dia']`, así que la noche queda idéntica por construcción. Más dos reglas de día en `login.component.scss` (el panel de la puerta de bodega pasa a blanco y el rechazo tiñe el mapa con `sepia(0.4) hue-rotate(-50deg) saturate(2.6)`; la vía del camión sigue blanca porque va sobre el botón principal).
- **Escena:** 0 líneas de TypeScript cambiadas (**observado**: `git diff` de `views/pages/registro/la-red/**` vacío). El lienzo WebGL se aclara con mezcla y filtros CSS (`mix-blend-mode: difference; filter: hue-rotate(180deg) saturate(1.5)`), el póster con `invert(1) hue-rotate(180deg)`, y el fondo vivo 2D se oculta.
- **Tarjeta y controles:** panel blanco al 90 %, campos con borde `#7d838c`, botón principal con el degradado oscurecido de la plataforma, logo oficial sin filtro, enlaces `#b4470b` (5,46:1).
- **Recuperar contraseña:** el título «Recupera tu contraseña» tuvo que dejar `--cui-heading-color` en `initial` fuera del shell (el tema lo remapea).
- **Equipos limitados:** `_registro-basico.scss` apaga las dos auroras del fondo; la escena sigue con su propio regulador de calidad ([Modo básico](modo-basico-y-movil-la-red.md)).
- **Pendientes:** medir el login claro con GPU real en móvil y, si cuesta, hacer la paleta clara **nativa** de la escena en TypeScript (M-381, decisión abierta); en claro, `/app/registercontact` pasa de azul marino a blanca (M-379); la landing pública no cambia entre modos (M-371). El diseño advertía que el giro de tono del rechazo podía verse verdoso sobre papel (**hipótesis**); el código lo resuelve con el filtro propio de claro descrito arriba (**observado**). `ENTREGA.md` dice que el login, recuperar y registro en claro se revisaron a ojo, pero no registra ese caso por separado.

---

## Pruebas unitarias

| Archivo | Qué cubre |
|---|---|
| `lg-interruptor-tema.component.spec.ts` y `tema.servicio.spec.ts` | 20 casos del componente y del servicio: modo inicial, alternar y quitar el atributo, `localStorage` que lanza, evento `storage` (válido, inválido, otra clave), que no toca el atributo ni la clave de CoreUI, que no hace peticiones, `role="switch"`, `aria-checked`, que dentro de un `<form>` no envía el formulario y las variantes |
| Specs de cabecera, menú móvil, login y registro | Integración del interruptor |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-11-22 | Iker Acevedo · redactado con Claude | Diseño y construcción del modo claro e interruptor (fase 2). |
| 2026-11-23 | Iker Acevedo · redactado con Claude | Primera versión de esta página. Commit `0c88ca39`: logo del botón de «Pedidos» visible en claro. |

---

## Observaciones

- El interruptor es lo único nuevo en los píxeles de noche (Iker debe aceptar esa excepción; el diseño la planteó así).
- El logo del menú lateral se pinta en blanco con `filter` y pierde el rojo de «Logi»; falta un PNG o SVG claro del logo, que es un activo de marca (M-354).
