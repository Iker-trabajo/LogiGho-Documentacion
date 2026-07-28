---

## Autor: Adalberto González
Fecha creacion: 2026-06-03
Estado: reemplazada

# ADR-001 — Arquitectura del módulo Guías de Devoluciones

**Autor:** Adalberto González
**Fecha:** 2026-06-03
**Estado:** Reemplazada por ADR-002

---

## Contexto

El módulo empezó siendo una pantalla de Power BI. Se decidió construirla en Angular para datos en tiempo real, extrayendo lógica a `FilterService` y `ChartComputerService`, con el componente manteniendo `datos[]` como fuente de verdad, sin signals.

---

## Opciones consideradas

### Opción A — Arquitectura por capas (services simples)

`FilterService` + `ChartComputerService`, sin reactividad automática.

**Pros:** cambios mínimos, sin dependencias nuevas, componente en ~280 líneas.
**Contras:** el cómputo de chart/tabla corría en el hilo principal; sin reactividad automática; no seguía el mismo patrón que otros dashboards migrados de Power BI.

---

## Decisión

**Se eligió:** Opción A (arquitectura por capas).

**Razón:** El módulo funcionaba correctamente y reducía el componente por capas sin tocar el flujo de carga ya probado.

---

## Consecuencias

**Positivas:** componente reducido de 890 a ~280 líneas; lógica de filtros y chart testeable de forma aislada.
**Negativas:** sin reactividad automática (`recalcular()` manual); cómputo pesado bloqueaba el hilo principal con el histórico completo en memoria.

---

## Impacto en el código

| Módulo / Repo  | Cambio                                                                                                          |
| --------------- | ---------------------------------------------------------------------------------------------------------------- |
| `SitioLogiGho` | Nuevos `models/`, `services/chart-computer.service.ts`, `services/filter.service.ts`, `workers/worker-utils.ts` |

---

## Historial de cambios

| Fecha      | Autor              | Cambio                                                |
| ----------- | -------------------- | ------------------------------------------------------ |
| 2026-06-03 | Adalberto González | Se implementó la Arquitectura por capas en todo el módulo |

---

## Referencias

- Reemplazada por [ADR-002](./ADR-002-guias-devolucion-store.md)
