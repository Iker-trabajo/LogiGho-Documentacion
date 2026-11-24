---

## Autor: Iker Acevedo (decisión) · redactado con Claude
Fecha creacion: 2026-11-23
Estado: aceptada, con una excepción declarada (Power BI)

# ADR-RED-003 — TypeScript nuevo solo en el armazón; la lógica de las vistas no se toca

**Autor:** Iker Acevedo (decisión) · redactado con Claude  
**Fecha:** 2026-11-23  
**Estado:** Aceptada. Una excepción fue pedida expresamente por Iker el 2026-11-23 (filtro de Power BI).

> Marcas: **observado** (ruta o fuente citada), **Iker** (fecha) o **hipótesis**.

---

## Contexto

Iker pidió «mucho cuidado con la lógica de todas las vistas» (2026-11-22). 30 de 53 vistas de logística y 25 de 44 de tienda leen el DOM desde TypeScript; un cambio de clase o de estructura puede romperlas. La spec fijó que el rediseño es HTML y SCSS, y TypeScript de producción solo para conectar presentación. La fase 2 (interruptor, indicador de navegación, precarga, modo básico) necesitaba lógica nueva.

---

## Opciones consideradas

### Opción A — Solo conectar presentación (regla original, §5.1)

TypeScript solo para imports y metadatos de las tres primitivas.

**Pros:** superficie mínima.  
**Contras:** el interruptor, la navegación y el modo básico necesitan estado y escuchas; no caben.

### Opción B — TypeScript nuevo solo en el armazón, con pruebas primero y justificación por archivo (elegida)

La enmienda de la fase 2 (`spec.md` §8) amplía la regla: TypeScript nuevo **solo en el shell** (interruptor, indicador de navegación, precarga, modo básico), con pruebas escritas antes y una fila por archivo en `excepciones-ts.md`. La lógica de las vistas sigue intocable. El script `aceptar.py` bloquea el commit de un slice que toque `.ts` de producción; las excepciones las hizo el coordinador a mano.

**Pros:** lo nuevo vive en `layout/` y `components/lg-*`, probado y aislado; ninguna vista cambia de comportamiento.  
**Contras:** más código de producción que mantener (23 archivos `.ts` de producción cambiados o nuevos frente a `main` en la cabeza actual; 17 en la verificación).

### Opción C — Permitir TypeScript en las vistas

**Pros:** resolvería de raíz cosas como el congelado al abrir modales (M-382).  
**Contras:** es justo lo que Iker pidió evitar; cada vista se auditaría una por una.

---

## Decisión

**Se eligió:** Opción B.

**Razón:** da las funciones nuevas sin tocar el comportamiento de ninguna vista. Resultado verificable: la escena de login y registro (`views/pages/registro/la-red/**`) tiene 0 líneas cambiadas; las plantillas existentes solo cambian `class`, el interruptor y el atributo `lgNavegacion`; la paridad de peticiones da 0 diferencias.

**Lo que contiene el TypeScript nuevo** (`excepciones-ts.md`):

| Archivo (bajo `src/app/`) | Qué es |
|---|---|
| `components/lg-estado`, `lg-encabezado-vista`, `lg-modal` | Primitivas de presentación, sin usar todavía en vistas |
| `components/lg-interruptor-tema/*` | Interruptor y `TemaServicio` (solo `data-lg-modo`, `localStorage` y `theme-color`) |
| `layout/default-layout/directives/lg-navegacion.directive.ts` | Atributos `data-lg-nav*` y clase `lg-destino` |
| `layout/default-layout/precarga/precarga-la-red.ts` y `app.config.ts` | Precarga selectiva |
| `layout/default-layout/rendimiento/*` | Modo básico |
| `layout/default-layout/default-layout.component.ts/.html`, `sidebar-interaccion.directive.ts`, imports de cabecera, menú móvil, login y registro | Conexión de lo anterior |
| `views/tienda/entregas/components/eg-select/eg-select.component.ts` | Solo las cadenas CSS de sus estilos (colores fijos a `var(--lg-…, <valor antiguo>)`) |

---

## Excepción: filtro de Power BI (2026-11-23)

A pedido expreso de Iker, el commit `0c88ca39` añade `shared/utils/filtros-tiendas-pbi.ts` y una línea `...filtrosTablasTiendaPbi(tiendas)` en cinco vistas (tienda: dashboard, informe de guías, rendimiento; dropshipping: dashboard; logística: sin despacho). Restringe el filtro «Ecosistema» a los ecosistemas de las tiendas del usuario. **Es lógica de vista**, a diferencia del resto del rediseño: no añade peticiones y con «Todas» el reporte sigue sin filtros. Se hizo en esta rama por decisión de Iker. **Sin verificar contra el reporte real** (M-387); `PR-preprod.md` aún no lo recoge.

---

## Consecuencias

**Positivas:** comportamiento de las vistas intacto salvo la excepción; cada pieza nueva se apaga con un cambio de una línea ([Rollback, nivel 3](../../../guias/rediseno-la-red-rollback-y-hotfix.md)).  
**Negativas:** el congelado al abrir ciertos modales o vistas con datos reales **no se arregla** (es lógica de cada vista; D44, M-382, M-386); la excepción de Power BI debe validarse en PreProd aparte.

---

## Impacto en el código

| Módulo / Repo | Cambio |
|---|---|
| `SitioLogiGho` | Los archivos de la tabla; 23 `.ts` de producción frente a `main` en la cabeza actual |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-11-23 | Iker Acevedo · redactado con Claude | Primera versión, con la excepción de Power BI. |

---

## Referencias

- [Sistema visual: cómo migrar o crear una vista](../../../frontend/sistema-visual-la-red.md#como-migrar-o-crear-una-vista)
