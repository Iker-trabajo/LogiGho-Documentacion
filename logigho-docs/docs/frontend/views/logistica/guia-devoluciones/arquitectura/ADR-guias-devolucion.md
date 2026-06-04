---

## Autor: Adalberto González
Fecha creacion: 2026-06-03  
Estado: Desarrollo

# ADR-001 — Arquitectura del módulo Guías de Devoluciones

**Autor:** Adalberto González  
**Fecha:** 2026-06-03  
**Estado:** Desarrollo

---

## Contexto

El módulo de Guías de Devoluciones empezó siendo una pantalla que mostraba información creada en Power BI.
Se decidió construirla directamente en la aplicación usando HTML y Angular el cual nos pueda brindar informacion de devoluciones en tiempo real.
Por esta razón, se analizaron alternativas para reorganizar la solución de una forma más clara y fácil de mantener.

---

## Arquitectura Por Capas (Dentro de el modulo)

Extraer lógica a servicios Angular simples (`FilterService`, `ChartComputerService`) y funciones puras compartidas (`worker-utils`). El componente mantiene el array `datos[]` como fuente de verdad y sigue orquestando la carga. Sin signals reactivos, sin persistencia offline.

**Pros:**
- Cambios mínimos sobre el comportamiento ya probado
- Sin nuevas dependencias externas
- El componente queda en ~280 líneas
- Cada servicio tiene una sola responsabilidad clara
- Riesgo bajo de regresiones

**Contras:**
- El componente sigue siendo la fuente de verdad del array de datos
- Sin reactividad automática — hay que llamar `recalcular()` manualmente tras cada cambio
- No escala bien si en el futuro se necesita compartir datos entre rutas

---


## Decisión

**Se eligió:** Arquitectura Por Capas (Dentro de el modulo)

**Razón:** El módulo funciona correctamente. Con lo que hicimos reduce el componente por capas, elimina duplicación entre workers y hace testeable la lógica de filtros y chart sin tocar el flujo de carga.

---

## Consecuencias

**Positivas:**
- Componente reducido de 890 a ~280 líneas — solo orquestación
- Lógica de filtros centralizada en `FilterService` — testeable de forma aislada
- Cómputo de chart centralizado en `ChartComputerService` — testeable de forma aislada
- Duplicación entre workers eliminada via `worker-utils.ts`
- Tipos centralizados en `devoluciones.models.ts` — fuente de verdad única

**Negativas:**
- Se debe llamar `recalcular()` manualmente tras cada cambio de filtro o datos — sin reactividad automática
- Si en el futuro se necesita compartir `datos[]` entre rutas, habrá que migrar al Signal Store

---

## Impacto en el código

| Módulo / Repo | Cambio |
|---|---|
| `SitioLogiGho` | 
Componente refactorizado. Nuevos archivos: `models/`, `services/`, `workers/worker-utils.ts`, `models/devoluciones.models.ts` 
`services/chart-computer.service.ts`, `services/filter.service.ts`, `workers/data-processor.worker.ts`, `workers/historico.worker.ts` 
|

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-06-03 | Adalberto González | Se implemento la Arquitectura por capas en todo el modulo. |

---