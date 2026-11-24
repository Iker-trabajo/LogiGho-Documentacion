---

## Autor: Iker Acevedo (decisión) · redactado con Claude
Fecha creacion: 2026-11-23
Estado: aceptada para noche; implementada y pendiente de aval para el claro; abierta para `prefers-color-scheme`

# ADR-RED-002 — Noche por defecto; claro por atributo y sin seguir la preferencia del sistema

**Autor:** Iker Acevedo (decisión) · redactado con Claude  
**Fecha:** 2026-11-23  
**Estado:** Aceptada (noche, Iker 2026-11-22). El modo claro lo pidió Iker el 2026-11-22 y está implementado, pendiente de aval en PreProd. Seguir `prefers-color-scheme` es una decisión **abierta**.

> Marcas: **observado** (ruta o fuente citada), **Iker** (fecha) o **hipótesis**.

---

## Contexto

Iker probó y aprobó la versión azul noche el 2026-11-22 (commit `3a531b16`, etiqueta `rediseno-la-red-noche-v1`) y pidió una versión clara de toda la plataforma con un interruptor junto a «Consultar guía», login y registro también en claro. La versión noche debía quedar **congelada**: añadir un segundo modo no podía mover nada de lo aprobado (D38).

---

## Opciones consideradas

### Opción A — Noche por defecto; claro con un atributo y un interruptor (elegida)

Noche = **sin atributo**. Claro = `html[data-lg-modo="dia"]`, guardado en `localStorage['lg-modo']` y aplicado por un script en línea de `index.html` antes del primer pintado.

**Pros:** la noche no cambia por construcción (todo lo claro cuelga del atributo); el primer pintado ya es el correcto, sin parpadeo; sin peticiones.  
**Contras:** quien prefiere claro en su sistema operativo ve noche la primera vez; hay que mantener dos paletas y el interruptor.

### Opción B — Mesa clara por defecto

Era la recomendación del agente de diseño (DD1), por el riesgo de texto ilegible sobre fondos blancos fijos.

**Pros:** menos riesgo con las vistas con colores fijos.  
**Contras:** contradice la dirección que Iker aprobó. El riesgo se neutraliza con la «mesa clara de lo no migrado» ([ADR-RED-001](ADR-RED-001-tema-por-tokens-y-atributo-raiz.md)), sin cambiar la dirección visual (D10).

### Opción C — Seguir `prefers-color-scheme` del sistema

**Pros:** respeta la preferencia de cada persona sin que toque nada.  
**Contras:** cambia lo que ve cada usuario sin que lo elija y obliga a validar los dos modos como predeterminados. **No se hizo; queda como alternativa a decidir** (`ENTREGA.md` §7).

---

## Decisión

**Se eligió:** Opción A.

**Razón:** es lo que Iker aprobó y lo único que garantiza que la noche no se mueve. El modo claro «Papel de ruta» (papel cálido, tinta azul marino) se diseñó como el registro de noche impreso de día; el diseño y la receta de arreglo de colores fijos están en [Modo claro](../../../frontend/modo-claro-la-red.md).

---

## Consecuencias

**Positivas:** 60 de 60 pares de contraste del modo claro y 27 de 27 del login llegan a su umbral; alternar el modo no hace peticiones (paridad 0 diferencias en noche y claro); el login, recuperar y registro en claro se resuelven con CSS sin una línea de la escena 3D.  
**Negativas:**

- El interruptor es lo único nuevo en los píxeles de noche (excepción que Iker debe aceptar).
- Hay modales que no se vieron en claro (M-380) y el login claro no se midió con GPU real (M-381).
- Cada arreglo de color fijo exige una de las dos formas de la receta (token con valor de noche idéntico o regla solo de día); un cambio descuidado puede mover la noche.
- `/app/registercontact` pasa a tarjeta blanca en claro (M-379) y la landing no cambia entre modos (M-371).

---

## Impacto en el código

| Módulo / Repo | Cambio |
|---|---|
| `SitioLogiGho` | `src/index.html` (script de pre-pintado y cargador claro), `components/lg-interruptor-tema/` (componente y `TemaServicio`), `scss/lared/_tokens.scss` (`lg-tokens-claro`, `--lg-shell-*`), `views/pages/registro/styles/_registro-claro.scss` |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-11-23 | Iker Acevedo · redactado con Claude | Primera versión. |

---

## Referencias

- [Modo claro e interruptor](../../../frontend/modo-claro-la-red.md)
