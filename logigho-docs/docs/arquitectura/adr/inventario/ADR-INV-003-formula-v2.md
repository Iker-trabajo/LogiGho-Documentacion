---

## Autor: Iker Acevedo  
Fecha creacion: 2026-10-10  
Estado: aceptada

# ADR-INV-003 — Fórmula v2 del consumo y el tránsito

**Autor:** Iker Acevedo  
**Fecha:** 2026-10-09 (INV-009) · aprobada 2026-10-10 (INV-036)  
**Estado:** Aceptada (aprobada por el CEO y el Jefe de Logística, vía Iker)

---

## Contexto

La fórmula anterior calculaba `Consumo = Ingresos − Actual`, lo que metía los ajustes de bodega (conteos) dentro del consumo de pedidos. Un ajuste de −3 dejaba el tránsito en −3 sin que ningún pedido hubiera salido (por ejemplo el producto 10845, −3.165).

## Opciones consideradas

### Opción A — Mantener v1

**Pros:** nada cambia para quien compara números.  
**Contras:** el tránsito se hunde con cada ajuste.

### Opción B — v2: `Consumo = Salidas − Anulaciones`

**Pros:** el consumo depende solo de movimientos de pedidos; el ajuste solo mueve `Actual`.  
**Contras:** quien comparaba cifras de antes y después verá una diferencia tras el go-live.

---

## Decisión

**Se eligió:** Opción B.

```
Actual      = Ingresos − Salidas + Anulaciones + Ajustes
Consumo     = Salidas − Anulaciones
En tránsito = Consumo − Entregas − Devoluciones − Pérdidas
```

**Razón:** los ajustes son conteo de bodega, no movimiento de pedido. Se sale a producción con `FORMULA_CONSUMO=v2`. Las devoluciones no suman a `Actual` solas (INV-004). Se eliminó el auto-ingreso (INV-030).

---

## Consecuencias

**Positivas:** el tránsito deja de hundirse por ajustes. VAL-3: un ajuste de −3 bajó `Actual` de 78 a 75 sin mover `Consumo` ni `En tránsito`; en el E2E el producto 10402 cuadra (`17 − 7 − 3 − 1 = 6`).  
**Negativas:** la variable por defecto del código sigue siendo `v1`; el valor `v2` depende de que Terraform lo fije en cada entorno (en producción ya lo hace).

---

## Impacto en el código

| Módulo / Repo | Cambio |
|---|---|
| `ApiLambdaInventario` | `CalculadoraResumen`, variable `FORMULA_CONSUMO` |
| Terraform `environments/prod/inventario` | `formula_consumo = "v2"` |
| Front | Pantalla Resumen y «¿Cómo se calcula?» |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-10-21 | Iker Acevedo | Documentado en MkDocs |

---

## Referencias

- [Fórmula v2 con ejemplo numérico](../../../backend/inventario/estados-y-formula.md#formula-v2)
