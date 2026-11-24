---

## Autor: Iker Acevedo (decisión) · redactado con Claude
Fecha creacion: 2026-11-23
Estado: aceptada (implementada y verificada en local; pendiente de PreProd)

# ADR-RED-004 — Sin View Transitions: la transición entre módulos se hace con atributos y CSS

**Autor:** Iker Acevedo (decisión) · redactado con Claude  
**Fecha:** 2026-11-23  
**Estado:** Aceptada por el equipo de construcción (D42, D43); pendiente de la prueba en iPhone real.

> Marcas: **observado** (ruta o fuente citada), **Iker** (fecha) o **hipótesis**.

---

## Contexto

Iker pidió (2026-11-22) una transición llamativa al cambiar de módulo (D43: «lo más llamativo posible»). La opción obvia del router de Angular, `withViewTransitions()`, **ya había roto el menú en iPhone**: en WebKit tumbaba la página al cargar y bloqueaba los toques del menú. `app.config.ts` de `origin/main` ya trae `withEnabledBlockingInitialNavigation()` sin ella y con el comentario del hotfix, y ningún estilo del proyecto usa `::view-transition` (**observado**). La causa de la sensación de «congelado» al cambiar de módulo es la espera de red del chunk de código y el bloqueo del hilo principal al crear la vista: no hay ninguna señal visible hasta que carga.

---

## Opciones consideradas

### Opción A — View Transitions del router

**Pros:** transición nativa entre estados.  
**Contras:** rompió el menú en WebKit (iPhone). Se descarta.

### Opción B — Indicador por atributos del router y CSS global (elegida)

Una directiva (`lgNavegacion`) refleja el estado del router en atributos `data-lg-nav*` del `.wrapper` y marca `lg-destino` en el enlace del menú; el CSS global (`_movimiento.scss`) dibuja: enlace destino al instante, contenido atenuado, la línea cálida de 2 px que ya existía en la cabecera como indicador, y la entrada coreografiada de la vista nueva. Todo con `transform` y `opacity`.

**Pros:** sin API nueva del navegador; la línea se anima en el compositor y sigue moviéndose aunque el hilo principal esté ocupado (**hipótesis** del diseño); cubre las 155 vistas sin editarlas (selector sobre `router-outlet + *`).  
**Contras:** los atributos recalculan el estilo del `.wrapper` (15–21 ms en una vista de 245 nodos con CPU ×4); por eso no se escriben en teléfono, tableta ni modo básico (M-384).

### Opción C — Retrasar la navegación para mostrar una animación de salida

**Pros:** salida visible del componente viejo.  
**Contras:** el router destruye la vista vieja al activar la nueva; animar su salida exigiría `@angular/animations` y retrasar la navegación. «Nunca se retrasa la navegación para mostrar una salida» (diseño). Se descarta.

---

## Decisión

**Se eligió:** Opción B. **No se reintroduce `withViewTransitions()`.**

**Razón:** funciona en WebKit por construcción (solo atributos y CSS) y mejora medido lo que importa: la primera respuesta visible al clic baja de 48–147 ms a 19–31 ms en escritorio y de 0,38–8,5 s a 0,04–0,28 s en un teléfono lento simulado.

---

## Consecuencias

**Positivas:** el menú responde al instante; sin riesgo de la API que rompió iPhone.  
**Negativas:**

- La transición **tapa** la espera, no la elimina: el bloqueo por inicializar vistas con datos reales sigue (D44).
- En teléfono lento, volver a un módulo ya visitado cuesta +33 a +82 ms: es el coste del código nuevo que corre en cada navegación; decisión abierta de Iker (quitar el indicador en teléfono).
- La línea de progreso con cargas de 120–300 ms se vacía y se rellena muy rápido (decisión abierta).
- **Pendiente:** probar en iPhone real que `:has()`, `translate` y `scale` sueltos funcionan como en Chromium.

---

## Impacto en el código

| Módulo / Repo | Cambio |
|---|---|
| `SitioLogiGho` | `layout/default-layout/directives/lg-navegacion.directive.ts`, `default-layout.component.html` (atributo `lgNavegacion` en `.wrapper`), `scss/lared/_movimiento.scss` |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-11-23 | Iker Acevedo · redactado con Claude | Primera versión. |

---

## Referencias

- [Movimiento y navegación entre módulos](../../../frontend/movimiento-y-navegacion-la-red.md)
