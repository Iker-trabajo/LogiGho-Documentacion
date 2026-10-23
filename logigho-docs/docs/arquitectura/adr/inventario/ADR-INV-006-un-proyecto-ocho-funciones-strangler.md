---

## Autor: Iker Acevedo  
Fecha creacion: 2026-10-09  
Estado: aceptada

# ADR-INV-006 — Un proyecto, varias funciones Lambda y migración Strangler Fig

**Autor:** Iker Acevedo  
**Fecha:** 2026-10-09 (INV-021, INV-022)  
**Estado:** Aceptada

---

## Contexto

Dos lambdas gemelas con la misma lógica copiada: cada arreglo se hacía dos veces y ya habían divergido. Había que reemplazarlas sin romper a los llamadores en producción.

## Opciones consideradas

### Opción A — Un proyecto, N funciones

**Pros:** cero duplicación; memoria, timeout, permisos y alarmas por función; una suite de pruebas; patrón ya usado en `ApiLambdaDevolucionesMasivo`.  
**Contras:** un cambio en el Dominio redespliega todas.

### Opción B — Una sola función con router interno

**Pros:** lo más simple.  
**Contras:** la trazabilidad (rápida, poca memoria) compartiría configuración con el proceso por carga (2 GB/15 min).

### Opción C — Núcleo compartido + lambdas separadas

**Pros:** útil si son canales distintos.  
**Contras:** aquí comparten casi todo; sería sobrearquitectura.

---

## Decisión

**Se eligió:** Opción A, migrando con **Strangler Fig**.

**Razón:** el sistema nuevo crece alrededor del viejo y se repuntan las **mismas rutas** el día del go-live (reversible en segundos). Primero lo que no reemplaza nada (trazabilidad, recalcular), luego el barrido en simulación, al final las rutas de escritura.

---

## Consecuencias

**Positivas:** los llamadores no cambian; rollback = repuntar la ruta a la gemela.  
**Negativas:** durante la semana de observación conviven ambos sistemas (las gemelas quedan con concurrencia 0 y sin ruta). Ocho funciones por mantener ([Arquitectura](../../../backend/inventario/arquitectura.md#un-proyecto-ocho-funciones)).

---

## Impacto en el código

| Módulo / Repo | Cambio |
|---|---|
| `LambdasLogiGho.Aplicacion/Logigho/Inventario/ApiLambdaInventario` | Proyecto nuevo (.NET 10, ReadyToRun) |
| `Terraform_Infraestructura_Logigho` | `modules/inventario`, `modules/inventario-rutas`, `environments/*/inventario` |
| `pipelines/inventario` | CI y CD de PreProd y producción |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-10-21 | Iker Acevedo | Documentado en MkDocs |

---

## Referencias

- [Arquitectura — Strangler Fig](../../../backend/inventario/arquitectura.md#strangler-fig-y-repunte-de-rutas)
