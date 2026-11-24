---

## Autor: Iker Acevedo (decisión) · redactado con Claude

Fecha creacion: 2026-11-23
Estado: desarrollo (verificado en local; pendiente de validar en PreProd)
Tipo: componente

# Modo básico y celular

**Servicio:** `RendimientoServicio` (`src/app/layout/default-layout/rendimiento/`)  
**Estilos:** `src/scss/lared/_basico.scss` (y `views/pages/registro/styles/_registro-basico.scss` para login y registro)  
**Detección temprana:** script en línea de `src/index.html`  
**Acceso:** Autenticado (shell) y público (login y registro)

!!! warning "Estado: verificado en local, pendiente de PreProd"
    Todo se probó en Chromium con emulación; **iPhone real y Safari real están pendientes** (el último hotfix de producción fue justo del menú en iPhone, por eso importa). Cifras en [Rediseño «La Red»](tema-la-red.md). Marcas: **observado** (archivo citado), **Iker (fecha)**, **hipótesis**.

---

## ¿Qué hace?

Dos cosas separadas:

1. **Teléfono y tableta** (pantalla de hasta 768 px o puntero táctil): el diseño es el completo, pero **sin animaciones**.
2. **Modo básico** (equipo limitado): el mismo diseño **sin animaciones y sin efectos caros de pintura**, para que la plataforma no se sienta colgada en un equipo que no da abasto.

Además, el rediseño corrigió el modo celular de las vistas (ver el final de esta página).

---

## Cuándo se activa el modo básico

El modo se escribe en `<html>` como `data-lg-rendimiento="basico"` y deja el motivo en `data-lg-rendimiento-motivo`. Se decide en dos momentos.

### 1. Antes del primer pintado (script de `index.html`)

Es la función `motivoTemprano()` de `rendimiento.deteccion.ts` escrita a mano, sin módulos (si cambia una, cambia la otra). Orden de decisión:

| Orden | Señal | Motivo | Notas |
|---|---|---|---|
| 1 | `localStorage['lg-rendimiento'] = 'completo'` | (ninguno) | Lo **prohíbe** todo: ni el script ni la medición lo activan |
| 2 | `localStorage['lg-rendimiento'] = 'basico'` | `preferencia` | Forzado a mano |
| 3 | `sessionStorage['lg-rendimiento-medido'] = 'basico'` | `medido` | Resultado de una medición previa en la misma sesión |
| 4 | Ahorro de datos del navegador (`saveData`) | `ahorro-datos` | Solo Chromium |
| 5 | Memoria del equipo ≤ 2 GB (`deviceMemory`) | `memoria` | Solo Chromium |
| 6 | ≤ 2 núcleos **y** el navegador expone la memoria | `nucleos` | Ver nota |

!!! note "Código frente a fuentes"
    `ENTREGA.md` resume la regla como «≤ 2 GB o ≤ 2 núcleos». En el código los núcleos **solo cuentan donde el navegador también expone la memoria** (Chromium): WebKit recorta `hardwareConcurrency` (un iPhone actual declara 2) y marcaría como limitado a todo iPhone (revisión R4, H18). Manda el código.

`prefers-reduced-motion` **no** activa el modo básico: habla de movimiento, no de la capacidad del equipo, y ya tiene su propio tratamiento (fundidos sin desplazamiento; decisión D51).

### 2. En ejecución (`RendimientoServicio`, desde el layout)

Se inicia con `this.rendimiento.iniciar()` en `default-layout.component.ts`. Mide lo que solo se sabe con el navegador en marcha:

1. Tras el primer `NavigationEnd`, 4 s después, en reposo y con la pestaña visible, **sonda de WebGL**: un contexto de 1×1 con `failIfMajorPerformanceCaveat`. Si falla, no hay WebGL o el renderizador es un rasterizador por software (`llvmpipe`, `softpipe`, `swrast`, «Basic Render»), pasa a básico al instante (motivos `webgl-software` o `sin-webgl`).
2. Si no, **1 s de cuadros** (`requestAnimationFrame`). Es «lenta» si la mediana supera 28 ms o más del 30 % de los cuadros pasa de 25 ms. Una ventana lenta se confirma con otra 1,5 s después; **solo dos lentas seguidas** activan el modo (motivo `cuadros`). Si la pestaña se oculta o un bloqueo de JavaScript (> 250 ms, más del 20 % de la ventana) domina la muestra, se descarta y se repite hasta 3 veces.
3. El resultado va a `sessionStorage` (no se mide en cada carga) y **nunca vuelve de básico a completo dentro de la sesión**.

No hay bucles permanentes ni peticiones de red. También apaga las animaciones de `@angular/animations` (grupos del menú, modales de CoreUI), que no son CSS, con `@.disabled` en `body`.

---

## Qué quita

| Nivel | Cuándo | Qué se quita |
|---|---|---|
| Sin movimiento | Modo básico **y** teléfono o tableta (`(max-width: 768px), (pointer: coarse)`) | Animaciones y transiciones del shell, las vistas y los overlays quedan en duración 0 y una iteración: el elemento aparece en su estado final desde el primer cuadro. Sin atenuado de la vista ni línea de progreso al navegar (el enlace destino sí se marca). Se **conservan** los indicadores de carga (un spinner congelado parece la aplicación colgada) y la barra del temporizador de SweetAlert |
| Sin efectos caros | **Solo** modo básico | `backdrop-filter`, sombras difusas grandes (quedan en un anillo de 1 px), retícula y degradado de fondo del shell (color plano), `scroll-behavior` suave. Las auroras del fondo del login y el registro |

Los indicadores de carga se reconocen por nombre de clase (`spinner`, `girando`, `cargando`, `loading`, `loader`, `shimmer`, `brillo`, `skel`, `esqueleto`, `.hueso`, `.fa-spin`…) o con la clase explícita `lg-indicador`. En teléfono el aspecto es el completo. Login, registro y landing quedan fuera de la regla de «sin movimiento»: la escena de «La Red» tiene su propio regulador (`registro/la-red/host/support.ts`), que baja de nivel si los cuadros no llegan.

---

## Soporte: forzarlo, desactivarlo, saberlo

Comandos para la consola del navegador de la persona (**observado** en `ENTREGA.md` §1 y en el código):

| Qué | Comando |
|---|---|
| Ver si está activo y por qué | `document.documentElement.dataset.lgRendimiento` y `document.documentElement.dataset.lgRendimientoMotivo` |
| Forzar el modo básico | `localStorage.setItem('lg-rendimiento','basico');location.reload()` |
| **Prohibirlo** (aunque el equipo sea limitado) | `localStorage.setItem('lg-rendimiento','completo');location.reload()` |
| Volver a la detección automática | `localStorage.removeItem('lg-rendimiento');sessionStorage.removeItem('lg-rendimiento-medido');location.reload()` |
| Volver a la versión azul si alguien dejó el modo claro | `localStorage.removeItem('lg-modo');location.reload()` |

La clave `completo` sirve a soporte para el caso inverso (un falso positivo) y no figura en `ENTREGA.md`; está en `rendimiento.deteccion.ts` y en el script de `index.html`. Esas tres claves (`lg-rendimiento`, `lg-rendimiento-medido` y `lg-modo`) son lo único que el rediseño deja en el navegador; la versión anterior del front las ignora.

---

## Límites conocidos

- **Una CPU lenta en reposo no se detecta** (M-378): sin nada que pintar, los cuadros salen a tiempo. Lo cubren a medias las señales de memoria y núcleos del script temprano.
- **Un navegador limitado a 30 cuadros por segundo** (ahorro de energía, escritorio remoto) **entra en modo básico durante la sesión**. Es deliberado (decisión D54); es una decisión abierta de Iker si prefiere subir el umbral.
- **Posibles falsos positivos** en equipos de 2 núcleos, con WebGL bloqueado o con pantalla a 30 Hz (**hipótesis** de la revisión R4, no medidos).
- **Dispositivos táctiles grandes** (tabletas, y cualquier equipo cuyo puntero principal se declare «grueso») quedan sin animaciones aunque la pantalla sea ancha: la consulta es `(max-width: 768px), (pointer: coarse)` (**observado** en el código; qué equipos de escritorio la cumplen no se probó).
- El rediseño no medía con una GPU real: el rendimiento del login claro con GPU en móvil sigue sin medirse (M-381).

---

## Celular: resumen de las correcciones

Auditoría a ojo de **148 rutas a 390 px** (`revisiones/movil/` en la spec): 9 defectos altos, 29 medios y 27 bajos. Se corrigieron los altos y medios que se resuelven con estilos (62 archivos SCSS, 7 commits). Resultado medido en la verificación final:

| Medición | Resultado |
|---|---|
| Scroll horizontal de página a 390 px, noche y claro | 130 rutas sin desborde, 26 que `main` desbordaba y ya no, **0 nuevas** |
| Scroll horizontal a 768 px | 155 sin desborde, 1 corregida, 0 nuevas |
| Tarjetas «Comunidades» y «Miembros» del muro | Sin corte ni doble scroll (la columna lateral ya no acompaña al scroll en escritorio: decisión abierta) |
| Cabecera | El criterio de la spec (CA17) pide que no desborde a 360, 375, 390 y 768 px; `ENTREGA.md` solo reporta los resultados de 390 y 768 px |

**Pendientes y límites del celular** (mejoras en el backlog; ninguna es nueva respecto de `main`):

| ID | Qué |
|---|---|
| M-375 | `app-tables`: el visor reserva 500 px aunque haya pocas filas y en teléfono solo caben una o dos columnas (exige TypeScript en la tabla compartida) |
| M-376 | En teléfono las tarjetas laterales del muro siguen ocultas con `display:none !important` desde `main`; decidir si deben verse |
| M-377 | `resumen-inventario` genera una página de ~21.000 px en teléfono: necesita paginación |
| M-374 | `estado-cuenta-mensajero` lanza un `TypeError` en cada detección de cambios (heredado de `main`) |
| M-355 | El corte de celular del layout (767,98 px) y el de las vistas (768 px) no coinciden: a 768 px exactos no hay barra lateral |
| M-373 | Wallet de tienda en móvil estrangulado: 13–17 cuadros perdidos al desplazar, también en `main` |
| (sin ID) | **Safari de iPhone:** solo hay capturas WebKit de escritorio emulado (M-362) |

---

## Pruebas unitarias

| Archivo | Qué cubre |
|---|---|
| `rendimiento.deteccion.spec.ts` y `rendimiento.servicio.spec.ts` | 33 casos según `excepciones-ts.md` (`ENTREGA.md` cuenta 34) del modo básico: señales tempranas, sonda de WebGL, ventanas de cuadros, descartes, persistencia y animaciones de Angular |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-11-23 | Iker Acevedo · redactado con Claude | Primera versión de esta página (modo básico, teléfono y correcciones de celular). |

---

## Observaciones

- La detección temprana de `index.html` y `rendimiento.deteccion.ts` son dos copias de la misma regla: cualquier cambio exige tocar las dos.
- El modo básico y el teléfono comparten el corte de pantalla con la directiva de navegación y con `sidebar-interaccion.directive.ts` (ver [Movimiento y navegación](movimiento-y-navegacion-la-red.md)).
