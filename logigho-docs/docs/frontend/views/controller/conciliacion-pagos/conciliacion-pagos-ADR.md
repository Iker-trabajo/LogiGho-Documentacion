---
autor: Adalberto González
fecha_creacion: 2026-04-29
estado: Desarrollo
---

# Implementación de patrón Strategy para transportadoras en módulo de conciliación de pagos

**Autor:** Adalberto González

**Fecha:** 2026-04-29

**Estado:** Desarrollo

---

## Contexto

El módulo gestiona pagos de 4 transportadoras legacy (Interrapidísimo, Envia, X-Cargo, Servientrega) con código duplicado. Se requería agregar D2E.

**Problema:** Cada transportadora requería ~400 líneas duplicadas (fetch, filter, export, totales) + componente de importación propio + validaciones.

**Restricciones:**
- Cero cambios a código en producción (legacy debe seguir funcionando)
- D2E se entrega en el ciclo actual
- Futuras transportadoras deben agregarse sin duplicación

---

## Opciones consideradas

### Opción A — Patrón legacy (duplicación)
Agregar D2E igual que las 4 existentes: variables propias, métodos propios, componente propio.

| Pros | Contras |
|------|---------|
| Sin curva de aprendizaje | ~400 líneas duplicadas por transportadora |
| Consistente con lo existente | Un bug = corregir en N componentes |
| Riesgo cero en producción | Con 10+ transportadoras: 3000+ líneas sin escalar |

### Opción B — Patrón Strategy para nuevas transportadoras
Implementar D2E con Strategy. Legacy coexiste hasta migración evaluada por el equipo.

| Pros | Contras |
|------|---------|
| Cero riesgo para código en producción | Dos patrones temporalmente |
| Escalable a 20+ transportadoras | Devs nuevos aprenden ambos estilos |
| Factory centraliza estrategias | Deuda técnica en legacy |
| Componente genérico reutilizable | — |

---

## Decisión

**Se eligió:** Opción B — Strategy Pattern

**Razón:** Entregar D2E sin riesgo en producción + escalabilidad para 6-8 transportadoras futuras. D2E valida el patrón; legacy se migra después si el equipo lo aprueba.

---

## Consecuencias

### Positivas
- Nueva transportadora = 1 strategy + 1 línea en factory + bloque HTML
- Bug fix en fetch/filter/export: un solo lugar
- Interfaz clara: `TransportadoraPagoStrategy` = contrato explícito
- Escalable a 20+ transportadoras

### Compromisos
- Dos patrones coexisten (legacy + Strategy)
- Variables legacy (`rows`, `rowsEnvia`, etc.) + `tablaEstadoMap`
- Deuda técnica: migración legacy pendiente

---

## Implementación

**Archivos creados:**
- `strategies/transportadora-pago.strategy.ts` — Interfaz
- `strategies/transportadora-pago.factory.ts` — Registry centralizado
- `strategies/transportadoras/d2e.strategy.ts` — Implementación D2E
- `components/importacion-pagos-generico/` — Componente reutilizable

**Cambios en componente:**
- Nuevas propiedades: `d2eStrategy`, `isD2EModalOpen`, `tablaEstadoMap`
- Nuevos métodos genéricos: `fetchTableDataByStrategy()`, `filterByStrategy()`, `getTotalMontoByStrategy()`, `exportarByStrategy()`

---

## Estado futuro

Se revisará si:
- Se migrarannlas transportadoras legacy a Strategy (decisión separada)
- Se alcanza 8+ transportadoras con Strategy (evaluar abstraer en clase base)
- Surge transportadora con flujo diferente (evaluar interfaces separadas)

