---

## Autor: Iker Acevedo (decisión) · redactado con Claude
Fecha creacion: 2026-11-23
Estado: aceptada (implementada y verificada en local; umbral de 30 cuadros por segundo abierto)

# ADR-RED-006 — Modo básico por detección temprana y sonda en ejecución

**Autor:** Iker Acevedo (decisión) · redactado con Claude  
**Fecha:** 2026-11-23  
**Estado:** Aceptada por el equipo de construcción (D51, D54); pendiente de validar en equipos reales.

> Marcas: **observado** (ruta o fuente citada), **Iker** (fecha) o **hipótesis**.

---

## Contexto

Las animaciones y los efectos de pintura (`backdrop-filter`, sombras difusas, retícula y degradado del shell) cuestan poco en un equipo bueno y mucho en uno limitado, sobre todo si el navegador renderiza por software. Iker pidió animaciones en las partes importantes, así que hacía falta una salida para los equipos que no las aguantan, sin penalizar a los demás.

---

## Opciones consideradas

### Opción A — Animaciones para todos

**Pros:** un solo comportamiento.  
**Contras:** un equipo limitado se siente colgado.

### Opción B — Apagar solo con `prefers-reduced-motion`

**Pros:** estándar.  
**Contras:** esa preferencia habla de **movimiento**, no de la capacidad del equipo, y ya tiene su propio tratamiento (fundidos sin desplazamiento). Se descarta como señal (D51).

### Opción C — Detección temprana barata más sonda en ejecución (elegida)

- **Antes del primer pintado** (script de `index.html`): preferencia forzada, resultado medido de la sesión, ahorro de datos, memoria ≤ 2 GB y ≤ 2 núcleos (solo donde el navegador expone la memoria).
- **En ejecución** (`RendimientoServicio`): sonda de WebGL por software o ausente y, si no, una ventana de 1 s de cuadros confirmada por una segunda; el resultado se guarda en `sessionStorage`.
- Teléfono y tableta: sin animaciones, con el aspecto completo.

**Pros:** lo barato actúa antes de pintar; lo caro solo se mide una vez por sesión; `localStorage['lg-rendimiento']` permite a soporte forzarlo o prohibirlo.  
**Contras:** una CPU lenta en reposo no se detecta (M-378); posibles falsos positivos.

---

## Decisión

**Se eligió:** Opción C.

**Razón:** cubre las señales que se pueden conocer sin esperar y las que solo se ven con el navegador en marcha, sin bucles permanentes ni peticiones. Detalle en [Modo básico y celular](../../../frontend/modo-basico-y-movil-la-red.md).

---

## Consecuencias

**Positivas:** equipos sin WebGL real, con ≤ 2 GB o con cuadros lentos mantienen el mismo diseño sin animaciones ni efectos caros; 34 casos de prueba (33 según `excepciones-ts.md`).  
**Negativas:**

- **Decisión abierta de Iker:** un navegador limitado a 30 cuadros por segundo (ahorro de energía, escritorio remoto) **entra en modo básico durante la sesión** (D54, mediana > 28 ms). Alternativa: subir el umbral para que conserve animaciones.
- Posibles falsos positivos en equipos de 2 núcleos, con WebGL bloqueado o con pantalla a 30 Hz (**hipótesis** de la revisión R4).
- Una vez en básico, no vuelve a completo dentro de la sesión.
- La regla vive duplicada: el script de `index.html` es la función `motivoTemprano()` escrita a mano; cambiar una exige cambiar la otra.
- Las fuentes resumen «≤ 2 núcleos» sin condición; el código solo la aplica si hay dato de memoria.

---

## Impacto en el código

| Módulo / Repo | Cambio |
|---|---|
| `SitioLogiGho` | `src/index.html` (script), `layout/default-layout/rendimiento/*`, `scss/lared/_basico.scss`, `views/pages/registro/styles/_registro-basico.scss`, `default-layout.component.ts` (`rendimiento.iniciar()`) |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-11-23 | Iker Acevedo · redactado con Claude | Primera versión. |

---

## Referencias

- [Modo básico y celular](../../../frontend/modo-basico-y-movil-la-red.md)
