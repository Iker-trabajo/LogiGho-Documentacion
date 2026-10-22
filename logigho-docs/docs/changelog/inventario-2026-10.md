# Liberación: nuevo sistema de Inventario (octubre 2026)

**Estado:** validado en PreProd (VAL-1/2/3, ensayo de go-live, E2E); salida a producción planeada. Producción **todavía no desplegada** a la fecha de esta entrada.

## Nuevas funcionalidades

- **`ApiLambdaInventario`** (.NET 10): 8 funciones Lambda en un proyecto: trazabilidad, movimientos por carga y por guía, barrido de cierre, recálculo del resumen y las tres de alertas diarias.
- **Idempotencia real** con clave única e índice único sparse `ux_clave`: reintentos y llamadas simultáneas no duplican.
- **Un cierre por guía**, nuevo movimiento **Pérdida** (`PerdidasInventario`) e **historial** de todo lo borrado o corregido (`HistorialMovimientosInventario`).
- **`GET /inventario/trazabilidad`**: guías de un producto con marcas de descuadre y Excel.
- **Barrido de cierre** (en simulación al inicio) y `verificar_invariantes`.
- **Observabilidad:** 22 alarmas, dashboard `Inventario-<entorno>` y tópico SNS.
- **Alertas diarias por Slack** (producto nuevo; PreProd, producción después): `/inventario/suscripciones`, `/inventario/despachos`.
- **Front:** trazabilidad, resumen con fórmula v2, despachos por día, administrar/activar inventario, bajo stock.

## Cambios que se notan

- **Fórmula v2:** los ajustes solo mueven `Actual`; el tránsito deja de hundirse.
- **Filtro de activación:** solo productos activados del proveedor (hoy 65 BTK) desde su primer ingreso. Productos nuevos no descuentan hasta activarse.
- **Sin auto-ingreso** desde `Productos.cantidad`: producto sin ingresos = 0.
- **Estados homologados** (tilde, mayúsculas, espacios); «Devolución Regional» y Archivada por fechas; estado desconocido = Revisar.
- **Retorno con cierre registrado:** se conserva y se marca.
- **Limpieza de datos única** en la ventana (~607 mil acciones en BTK), con respaldo y rollback exacto.
- **Rutas:** `actualizaInventarioLotes` y `actualizarInventario` se repuntan a las funciones nuevas en el REST `7elgyten88`; sin cambios para los llamadores. CORS `*`.
- **Infraestructura:** Terraform (`modules/inventario`) y Azure DevOps (`Lambdas-Inventario-CD-*`); CloudFormation retirado.

## Pendientes conocidos

- Partir lotes en Liquidaciones, validación de stock al crear guías, ingresos/ajustes idempotentes, «despachado» real, causas raíz de las anomalías, aislar PreProd de Inter, access logs, notificador en producción y retirar el puente `g3iz6qk3f0`: ver [Tareas futuras](../backend/inventario/tareas-futuras.md).
- Discrepancias entre README, runbook y código: ver [Operación](../guias/inventario-operacion.md#discrepancias-conocidas-entre-documentos-y-codigo).

## Documentación

- [Visión del dominio](../backend/inventario/index.md), [Arquitectura](../backend/inventario/arquitectura.md), [Estados y fórmula](../backend/inventario/estados-y-formula.md), [Llamadores](../backend/inventario/llamadores.md), [Alertas](../backend/inventario/alertas-slack.md), [API](../api/inventario.md), [Operación](../guias/inventario-operacion.md), [ADR](../arquitectura/adr/inventario/index.md), [Pantallas del front](../frontend/views/logistica/inventario/inventario-front.md).
