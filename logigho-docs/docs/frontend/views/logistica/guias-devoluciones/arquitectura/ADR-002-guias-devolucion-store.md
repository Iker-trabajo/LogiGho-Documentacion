## ADR-002: Migración a Store + Repository + Workers

**Autor:** Adalberto González
**Fecha:** 2026-07-28
**Estado:** aceptada

---

## Contexto

El ADR-001 estableció una arquitectura por capas simple (`FilterService`+`ChartComputerService`), con el cómputo de chart/tabla corriendo en el hilo principal. Al migrar otros dashboards de Power BI (`dashboard-sin-despacho`) se estableció un patrón estándar para la plataforma: Store con signals + Repository + Web Workers.

---

## Decisión

**Se eligió:** migrar a Store + Repository + Rules + Workers, alineando el módulo con `dashboard-sin-despacho` y `relacion-despacho`.

**Razón:** sacar el cómputo pesado del hilo principal (crítico con el histórico completo en memoria) y estandarizar el patrón entre todos los dashboards migrados.

---

## Consecuencias

**Positivas:** cómputo de chart/tabla movido a `agregacion.worker`; bug de header corregido de paso; histórico reducido de 8 a 4 meses; `&campos=` reduce el peso de cada página.
**Negativas:** mayor curva de entrada para tocar el módulo; requiere entender el contrato de mensajes del Worker.

---

## Impacto en el código

| Módulo / Repo | Cambio |
|---|---|
| `SitioLogiGho` | Eliminados `services/`. Nueva carpeta `helpers/` con `devoluciones.store.ts`, `devoluciones.repository.ts`, `devoluciones.rules.ts`, `agregacion.worker.ts` y los workers ya existentes |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-28 | Adalberto González | Migración completa a Store + Repository + Workers |

---

## Referencias

- Reemplaza a [ADR-001](./ADR-guias-devolucion.md)
