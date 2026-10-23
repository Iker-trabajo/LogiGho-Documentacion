---

## Autor: Iker Acevedo  
Fecha creacion: 2026-10-09  
Estado: aceptada

# ADR-INV-001 — Clave única de movimiento e índice único sparse

**Autor:** Iker Acevedo  
**Fecha:** 2026-10-09 (INV-010 e INV-029)  
**Estado:** Aceptada (INV-029 se verifica en DocumentDB con `--probar-sparse` en D-7 del go-live)

---

## Contexto

Las lambdas viejas evitaban duplicados con «¿existe? → si no, inserto»: dos pasos separados. Dos ejecuciones casi simultáneas (reintentos tras un 504, el cron de liquidación con una carga) veían «no existe» y ambas insertaban. Se midieron devoluciones duplicadas (−94.560 u). Las banderas de `PedidosInter` se marcaban después, así que no protegían.

## Opciones consideradas

### Opción A — Índice único normal sobre la clave

**Pros:** garantía atómica en el motor.  
**Contras:** exige que *todo* el dato esté sin duplicados, incluidos los productos inactivos que no se van a limpiar.

### Opción B — Índice único + sparse sobre `clave` (`guía|producto|slot|TIPO`)

**Pros:** solo indexa los documentos que llevan `clave`; los viejos o de productos inactivos no lo bloquean; las gemelas siguen funcionando (insertan sin `clave`).  
**Contras:** en Atlas dos `clave:null` chocan, así que el sistema nunca debe escribir `clave:null` (el verificador lo vigila); hay que comprobar que DocumentDB acepta único + sparse.

### Opción C — Seguir con consultar-luego-insertar y más banderas

No resiste la concurrencia. Descartada.

---

## Decisión

**Se eligió:** Opción B.

**Razón:** la verificación y la escritura ocurren dentro del motor en una sola operación atómica. Un insert repetido recibe `E11000`, que el código trata como «ya existía» (éxito idempotente). El `slot` en la clave permite dos movimientos legítimos del mismo producto en dos casillas. Las banderas de `PedidosInter` quedan solo como optimización de lectura y se marcan **al final**.

---

## Consecuencias

**Positivas:** repetir o simultanear una escritura no puede duplicar (8 ejecuciones simultáneas sin duplicados). Los reintentos son seguros para pedidos.  
**Negativas:** exige limpiar duplicados y «sellar» (`slot`/`clave`) antes de crear `ux_clave`. Los ingresos y ajustes manuales no tienen clave natural y siguen sin ser idempotentes ([Tareas futuras](../../../backend/inventario/tareas-futuras.md#ingresos-y-ajustes-idempotentes)). La guía no puede contener `|`.

---

## Impacto en el código

| Módulo / Repo | Cambio |
|---|---|
| `LambdasLogiGho` (`ApiLambdaInventario`) | `ClaveMovimiento`; repositorio de escritura con insert y captura de `E11000` |
| Base de datos | `ux_clave` único + sparse en las 5 colecciones (go-live P6) con `indices_golive.py` |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-10-21 | Iker Acevedo | Documentado en MkDocs |

---

## Referencias

- [Arquitectura — flujo por guía](../../../backend/inventario/arquitectura.md#flujo-por-guia)
- [Operación — P6 índices únicos](../../../guias/inventario-operacion.md)
