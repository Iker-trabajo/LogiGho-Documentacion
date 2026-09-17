## Autor: Iker Acevedo
Fecha creación: 2026-09-16
Estado: preprod
Tipo: componente

# Auditoría de Excepciones

**Selector:** `app-auditoria-excepciones`
**Ubicación:** `src/app/views/director-de-operaciones/costos-transportadora/components/auditoria-excepciones`

---

## ¿Qué hace?

Lista las guías que el motor Dinámico **intentó** liquidar y no pudo, con el motivo exacto (`MotivoNoLiquidacion`) y un botón directo para ir a la pestaña donde se corrige ese motivo — sin que el usuario tenga que adivinar dónde queda cada configuración.

Incluye 2 KPIs (guías pendientes de resolver, motivo más frecuente) y filtro por texto (guía, tienda o motivo) y por rango de fechas.

---

## Por qué hoy siempre está vacía — y no es un bug

Lee de la colección `AuditoriaLiquidaciones`, que **todavía no existe**: la capa que persiste un intento fallido de liquidación es infraestructura pendiente (ver [fases del proyecto](../../../../../backend/lambdas-dotnet/lambdas/ApiLambdaLiquidacionesLogighoAOT/motor-dinamico/orquestador.md#fases-del-proyecto), F3/F4). La pantalla se construyó completa y conectada igual, para que el día que exista ese escritor, conectarla sea solo la consulta — no rediseñar la pantalla. El componente no oculta la tabla vacía sin explicación; un aviso visible cuenta por qué está así.

---

## Se eliminó el KPI de "Impacto Estimado en $"

Una versión anterior mostraba un monto estimado de plata perdida por las guías sin liquidar. Se quitó porque **ni `PedidoLiquidable` ni `MotivoNoLiquidacion` traen un valor monetario** — ese número era inventado. Mostrar una cifra de plata que nadie calculó de verdad es exactamente el tipo de dato que este módulo, tratándose de liquidaciones reales, no se puede dar el lujo de inventar.

---

## `irAConfigurar()`

Usa `MOTIVO_NO_LIQUIDACION_TAB` (ver [Modelos](../modelos/liquidacion-transportadora-models.md#motivo_no_liquidacion_tab)) para saltar directo a la pestaña correcta según el motivo de la excepción — Trayectos, Tarifas o Configuración General. Si el motivo es un error operativo puntual (sin pestaña que lo resuelva), el botón no aparece.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-09-16 | Iker Acevedo | Documentación inicial. |
