# Decisiones de Inventario (ADR)

Decisiones clave del sistema [`ApiLambdaInventario`](../../../backend/inventario/index.md), en el formato de la [plantilla ADR](../../../plantillas/PLANTILLA-ADR.md). Las decisiones originales (INV-001…INV-042) viven en el registro del dominio; aquí están las que definen el diseño.

| ADR | Decisión | Decisiones de origen |
|---|---|---|
| [ADR-INV-001](ADR-INV-001-clave-unica-e-indice-sparse.md) | Clave única + índice único sparse | INV-010, INV-029 |
| [ADR-INV-002](ADR-INV-002-un-cierre-por-guia.md) | Un solo cierre por guía y producto | INV-008, INV-035 |
| [ADR-INV-003](ADR-INV-003-formula-v2.md) | Fórmula v2 del consumo y el tránsito | INV-009, INV-036 |
| [ADR-INV-004](ADR-INV-004-filtro-de-activacion.md) | Filtro de activación por proveedor | INV-003, INV-034 |
| [ADR-INV-005](ADR-INV-005-pedidosinter-fuente-de-verdad-y-retorno.md) | `PedidosInter` como fuente de verdad; el retorno con cierre se conserva | INV-035, INV-038, INV-039 |
| [ADR-INV-006](ADR-INV-006-un-proyecto-ocho-funciones-strangler.md) | Un proyecto, varias funciones, migración Strangler Fig | INV-021, INV-022 |
| [ADR-INV-007](ADR-INV-007-cors-abierto.md) | CORS abierto (`*`) | decisión de Iker 2026-10-19 |

## Decisiones de origen

Referencias usadas en estas páginas (todas aprobadas por Iker salvo que se indique):

| ID | Resumen |
|---|---|
| INV-001 / INV-028 | Alcance inicial: ecosistema BTK, solo productos activos |
| INV-002 | Sincronía vía liquidación + barrido de cierre |
| INV-003 | Activación por banderas `Tienda.ServicioInventario` y `Producto.ManejaInventario` |
| INV-004 | Devoluciones: ingreso manual a bodega tras validar |
| INV-021 | Migración incremental (Strangler Fig) |
| INV-023 | Native AOT no es obligatorio; ReadyToRun |
| INV-024 / INV-025 | Índice de trazabilidad; roles de trazabilidad (`Jefe Logistica`, `Administrador`, `Desarrollador`, `CEO`) |
| INV-026 | Fechas de `PedidosInter` en `America/Bogota` por ahora; pendiente `Tienda.ZonaHoraria` |
| INV-027 | La limpieza se ejecuta una sola vez en el go-live |
| INV-030 | Se elimina el auto-ingreso |
| INV-037 | Salida a producción como MVP |
| INV-041 / INV-042 | Decisiones de construcción de las olas 1c y 2, a ratificar |
