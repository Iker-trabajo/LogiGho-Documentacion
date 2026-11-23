---

## Autor: Iker Acevedo (decisión) · redactado con Claude
Fecha creacion: 2026-11-23
Estado: aceptada (implementada y verificada en local; pendiente de PreProd)

# ADR-RED-001 — El tema es una capa de tokens `--lg-*` activada por un atributo raíz

**Autor:** Iker Acevedo (decisión) · redactado con Claude  
**Fecha:** 2026-11-23  
**Estado:** Aceptada. Iker aprobó la dirección visual el 2026-11-22; esta forma de montarla es decisión del equipo de construcción (S1, D5), pendiente del aval final en PreProd.

> Marcas: **observado** (ruta o fuente citada), **Iker** (fecha) o **hipótesis**.

---

## Contexto

Había que llevar la identidad del login y del registro a 332 superficies del front (230 vistas, 89 componentes, 7 compartidos, estados y overlays) **sin cambiar datos, permisos, cálculos ni peticiones** (Iker, 2026-11-22: «mucho cuidado con la lógica de todas las vistas»). El punto de partida tenía 8.455 colores hexadecimales fijos en SCSS y plantillas, 841 `!important` y 197 estilos en línea (cifras del diseño `tokens-y-sistema.md`). Además había que poder volver atrás y no cerrar la puerta al [Sistema de Experiencias](../../../frontend/experiencias/index.md), que es una propuesta aún no aprobada.

---

## Opciones consideradas

### Opción A — Capa de tokens activada por un atributo en `<html>` (elegida)

Variables `--lg-*` definidas bajo `:root[data-lg-tema='la-red']`; las vistas migradas solo consumen tokens, utilidades `lg-*` y mixins `lg.*`.

**Pros:** un solo punto de activación; el modo claro sale casi gratis porque las vistas solo consumen tokens (D12); no hay que tocar la lógica; el remapeo de CoreUI hereda sin tocar plantillas.  
**Contras:** las vistas migradas **no funcionan sin el atributo**, así que quitarlo no revierte nada; hay que migrar a mano las vistas con fondos fijos, y las que no se migren quedan en «mesa clara».

### Opción B — Estilos propios por vista, sin capa común

Reescribir el SCSS de cada vista con sus colores.

**Pros:** sin dependencia de un atributo global.  
**Contras:** repite la situación de partida (8.455 hex), no permite modo claro ni cambios de paleta sin recorrer cada archivo y no es verificable de forma uniforme.

### Opción C — Montarlo dentro del Sistema de Experiencias

Usar `data-experiencia` y los nombres de token de esa propuesta.

**Pros:** un único mecanismo si la propuesta se aprueba.  
**Contras:** el front de experiencias aún no existe en la base (D5) y la propuesta no está aprobada. **No se descartó:** la opción A «queda compatible» (D5). Hay nombres de token que coinciden (`--lg-texto`, `--lg-superficie`, `--lg-borde`) y sus valores no están reconciliados (**observado**).

---

## Decisión

**Se eligió:** Opción A.

**Razón:** reversión de un solo punto de *activación* (aunque no de un solo punto de *reversión*: ver consecuencias), cobertura máxima con ediciones mínimas y compatibilidad con la propuesta de Experiencias (D5).

**Decisiones de detalle que la acompañan:**

- **Mesa clara de lo no migrado** (S3, D11, D17): el contenedor de la vista (`.wrapper .body > .container`) vale tokens de día por defecto y pasa a noche cuando la vista lleva `lg-vista-noche` en su raíz, detectado con `:has()`. Primero se probó una lista generada de vistas pendientes; la revisión R1 demostró dos fallos (selectores compartidos y `display` sobre hosts) y se cambió por el marcador en la plantilla. Sin `:has()`, la vista degrada a mesa clara (legible).
- **Especificidad baja** (S9, D13): el remapeo de CoreUI pone solo el prefijo del tema en `:where()`; gana a CoreUI por orden y **pierde** contra el estilo de cada componente Angular.
- **Aislamiento de La Red:** los tokens interiores no importan nada de `views/pages/registro/la-red`, para no arrastrar Three.js al bundle interior (CA10).

---

## Consecuencias

**Positivas:** misma identidad en toda la plataforma; modo claro e impresión por tokens; ninguna vista queda ilegible en ningún commit; los cambios de paleta futuros son de un archivo.  
**Negativas:**

- **Quitar `data-lg-tema` no es un rollback válido** (`ENTREGA.md` §8). La reversión es por commit o por despliegue ([Rollback](../../../guias/rediseno-la-red-rollback-y-hotfix.md)). `spec.md` §7 decía lo contrario; manda `ENTREGA.md`.
- 581 avisos de Sass por declaraciones tras reglas anidadas en el build (`main`: 18); no bloquean hoy pero el orden de la cascada cambiará al subir de versión de Sass (M-385).
- SweetAlert2 queda claro en los dos modos (79 llamadas con colores en línea; D18, M-358).

---

## Impacto en el código

| Módulo / Repo | Cambio |
|---|---|
| `SitioLogiGho` | `src/index.html` (atributo), `src/scss/_lared-tema.scss` y `src/scss/lared/*`, 343 SCSS y 256 plantillas (solo clases) migrados |
| `LambdasLogiGho`, Terraform | Ninguno |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-11-23 | Iker Acevedo · redactado con Claude | Primera versión. |

---

## Referencias

- [Rediseño «La Red»](../../../frontend/tema-la-red.md) y [Sistema visual y tokens](../../../frontend/sistema-visual-la-red.md)
- [Sistema de Experiencias: tokens y estilos](../../../frontend/experiencias/tokens-y-estilos.md) (propuesta)
