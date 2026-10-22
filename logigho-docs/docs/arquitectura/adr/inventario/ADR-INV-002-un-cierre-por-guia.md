---

## Autor: Iker Acevedo  
Fecha creacion: 2026-10-09  
Estado: aceptada

# ADR-INV-002 — Un solo cierre por guía y producto

**Autor:** Iker Acevedo  
**Fecha:** 2026-10-09 (INV-008) · 2026-10-10 (INV-035)  
**Estado:** Aceptada

---

## Contexto

Una guía puede terminar en entrega, devolución, anulación o pérdida. En los datos había guías con dos cierres a la vez (anulada + entrega, devolución + entrega), que producían tránsito negativo, porque nadie decidía cuál era el verdadero. En la plataforma hay guías «anuladas» que sí se entregaron.

## Opciones consideradas

### Opción A — Permitir varios cierres

**Pros:** no borra nada.  
**Contras:** la fórmula del tránsito se vuelve ambigua.

### Opción B — Exactamente un cierre por guía × producto; la evidencia física gana

**Pros:** el libro es coherente con `PedidosInter` y el tránsito cuadra.  
**Contras:** borra cierres (con historial).

---

## Decisión

**Se eligió:** Opción B.

**Razón:** `ReglaCierreUnico` decide el cierre por el estado homologado del pedido, con la evidencia física por encima: Entregada y Pagada → Entrega (Pagada aunque no tenga `Fecha Entrega`; si además tiene devolución, se borra la devolución); Anulada con `Fecha Entrega` → Entrega; Devolución completada → Devolución (una Entrega sobrante se borra); indemnizada, siniestrada, facturada o error → Pérdida; estado desconocido → no se toca. Todo borrado deja el original en `HistorialMovimientosInventario`.

---

## Consecuencias

**Positivas:** sin cierres dobles; la trazabilidad marca `CIERRE_DOBLE` y `CIERRE_FALTANTE` con una sola fuente de verdad.  
**Negativas:** depende de que el estado de `PedidosInter` sea correcto; en retorno con cierre registrado la regla se suspende ([ADR-INV-005](ADR-INV-005-pedidosinter-fuente-de-verdad-y-retorno.md)).

---

## Impacto en el código

| Módulo / Repo | Cambio |
|---|---|
| `ApiLambdaInventario` | `ReglaCierreUnico`, `ClasificadorEstado`, `Reconciliador` |
| Limpieza de datos | Reglas R3, R4 y R5 |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-10-21 | Iker Acevedo | Documentado en MkDocs |

---

## Referencias

- [Estados, cierres y fórmula](../../../backend/inventario/estados-y-formula.md#un-cierre-por-guia)
