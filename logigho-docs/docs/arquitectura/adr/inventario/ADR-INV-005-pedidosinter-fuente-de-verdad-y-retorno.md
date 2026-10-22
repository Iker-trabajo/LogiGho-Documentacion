---

## Autor: Iker Acevedo  
Fecha creacion: 2026-10-10  
Estado: aceptada

# ADR-INV-005 — `PedidosInter` es la fuente de verdad; el retorno con cierre se conserva

**Autor:** Iker Acevedo  
**Fecha:** 2026-10-10 (INV-035, INV-038, INV-039)  
**Estado:** Aceptada

---

## Contexto

Los movimientos de inventario deben reflejar lo que dice el pedido. Pero 346 guías BTK están en estado de retorno con un cierre ya registrado (entrega, devolución, anulación). Algunas se explican por procesos externos defectuosos (`ProcesoGuiasArchivadas`, `gestion-guias`) y no por datos malos. Un reconciliador que borre cualquier cierre que el estado no pida destruiría movimientos reales.

## Opciones consideradas

### Opción A — Borrar el cierre que el estado de retorno no pide

**Pros:** el libro siempre coincide con el estado.  
**Contras:** si la entrega o devolución era real, se pierde un movimiento.

### Opción B — Conservar el cierre y marcarlo

**Pros:** nunca se destruye evidencia con un estado ambiguo.  
**Contras:** el desglose del tránsito necesita un campo extra (`RetornoConCierre`); queda trabajo de revisión para desarrollo.

---

## Decisión

**Se eligió:** Opción B.

**Razón:** `PedidosInter` manda: la guía sigue en retorno, esperando llegar a bodega. El reconciliador **no borra, no corrige ni inserta** cierres (la salida sí se reconcilia). La trazabilidad marca `RETORNO_PENDIENTE_BODEGA` (cierre = Devolución) o `ESTADO_CONTRADICTORIO` (otro cierre). Cuando pasa a «Devolución completada» se reconcilia normal. Reglas relacionadas: Pagada = Entrega y la devolución sobrante se borra con historial (INV-035); «Devolución Regional» de Inter se decide por fechas y, sin fechas y con cierre, se trata como retorno (INV-039).

---

## Consecuencias

**Positivas:** el go-live no destruye movimientos de guías en retorno.  
**Negativas:** el tránsito de esas guías sigue restando sus cierres, así que `EnTransito = PorDespachar + Admitida + TransitoIda + (TransitoRetorno − RetornoConCierre) + SinClasificar`. Hasta corregir `ProcesoGuiasArchivadas` y `gestion-guias` ([Tareas futuras](../../../backend/inventario/tareas-futuras.md#causas-raiz-de-las-anomalias)), el barrido en escritura sigue bloqueado.

---

## Impacto en el código

| Módulo / Repo | Cambio |
|---|---|
| `ApiLambdaInventario` | `ClasificadorEstado.ConservaCierres`, `Reconciliador`, `DetectorDescuadre`, `RecalcularResumen` |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-10-21 | Iker Acevedo | Documentado en MkDocs |

---

## Referencias

- [Estados, cierres y fórmula](../../../backend/inventario/estados-y-formula.md#guias-en-retorno-con-cierre-ya-registrado-inv-038)
