---

## Autor: Adalberto González
Fecha creacion: 2026-07-28
Estado: aceptada

# ADR-002 — Migración a Store + Repository + Workers

**Autor:** Adalberto González
**Fecha:** 2026-07-28
**Estado:** Aceptada

---

## Contexto

Al migrar otros dashboards de Power BI (`dashboard-sin-despacho`) se estableció un patrón estándar para la plataforma: Store con signals + Repository + Web Workers. La arquitectura anterior de este módulo (ADR-001, `FilterService`+`ChartComputerService`) corría el cómputo de chart/tabla en el hilo principal, lo cual podía bloquear la UI con el histórico completo en memoria.

---

## Opciones consideradas

### Opción A — Mantener services simples (ADR-001)

Seguir con `FilterService`/`ChartComputerService`.

**Pros:** sin migración.
**Contras:** cómputo en hilo principal; inconsistente con el resto de dashboards migrados.

### Opción B — Store + Repository + Rules + Workers

Alinear con `dashboard-sin-despacho` y `relacion-despacho`: `DevolucionesStore` (signals), `DevolucionesRepository` (HTTP), `devoluciones.rules.ts` (funciones puras sin clase) y `agregacion.worker.ts` (cómputo en Web Worker).

**Pros:** cómputo fuera del hilo principal; estado reactivo sin `recalcular()` manual; mismo patrón en toda la plataforma; deduplicación O(1) vía `Map`.
**Contras:** migración de mayor alcance; requiere entender el contrato de mensajes del Worker.

---

## Decisión

**Se eligió:** Opción B.

**Razón:** Estandarizar todos los dashboards migrados de Power BI bajo un mismo patrón, y sacar el cómputo pesado del hilo principal.

---

## Consecuencias

**Positivas:** cómputo de chart/tabla movido a `agregacion.worker`; bug de header `headersecurity`/`headerSecurity` corregido de paso; histórico reducido de 8 a 4 meses; `&campos=` reduce el peso de cada página.
**Negativas:** mayor curva de entrada para tocar el módulo; un cambio en el cómputo requiere modificar `agregacion.worker.ts`, que corre sin acceso a DOM/`window`.

---

## Impacto en el código

| Módulo / Repo  | Cambio                                                                                                                                                          |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `SitioLogiGho` | Eliminados `services/`. Nueva carpeta `helpers/` con `devoluciones.store.ts`, `devoluciones.repository.ts`, `devoluciones.rules.ts`, `agregacion.worker.ts` y los workers ya existentes |

---

## Historial de cambios

| Fecha      | Autor              | Cambio                                                                    |
| ----------- | -------------------- | ---------------------------------------------------------------------- |
| 2026-07-28 | Adalberto González | Migración completa a Store + Repository + Workers, alineado con `dashboard-sin-despacho` |

---

## Referencias

- Reemplaza a [ADR-001](./ADR-guias-devolucion.md)
