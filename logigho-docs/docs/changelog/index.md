# Hisotial de cambios

Historial de cambios, nuevas funcionalidades y correcciones del sistema LogiGho.

---

## Formato de entrada

```
## [version o fecha] — Descripción corta

### Nuevas funcionalidades
- Descripción de qué se agregó

### Cambios
- Descripción de qué cambió

### Correcciones
- Descripción de bugs corregidos

### Documentación
- Qué se documentó en esta entrega
```

---

## [2026-07-27] — Pancake: doble escritura de páginas, fix de ventana y endpoint on-demand

### Nuevas funcionalidades

- **Endpoint on-demand** `ApiLambdaConsultarEstadisticasPagina` (API Gateway `POST`): trae estadísticas frescas de una página (hoy / ayer / rango personalizado) **sin persistir**, con fallback de token de página.
- `ApiLambdaListarPaginasPancake` ahora hace **doble escritura** Mongo + Aurora MySQL.

### Correcciones

- Ventana `cierre_dia_anterior`: `until` corregido de `00:00:00` del día siguiente a `**23:59:59`** del mismo día — Pancake ya no arrastra gasto del otro día.

### Documentación

- Nueva página **Endpoint on-demand (API)** en Backend → Integración Pancake; actualizadas las páginas de `ListarPaginasPancake` (SQL) y `CalcularVentanaTiempo` (fix `until`).

---

## [2026-07-13] — Integración Pancake (estadísticas de campañas)

### Nuevas funcionalidades

- Pipeline serverless de **5 lambdas .NET 8** que recolecta las estadísticas de campañas de Pancake (`pages.fm`) 4 veces al día (7am/9am/2pm/5pm hora Colombia), orquestado con **AWS Step Functions** y agendado con **EventBridge Scheduler**.
- **Doble escritura** de estadísticas por campaña: MongoDB + **RDS Aurora MySQL** (patrón Composite, best-effort, UPSERT idempotente, modo inerte plug-and-play).

### Documentación

- Nueva sección **Backend → Integración Pancake**: visión general con diagrama de arquitectura, una página por lambda, orquestación (Step Functions + ASL + EventBridge) y guía de operación (ejecución manual, agregar franjas, monitoreo y troubleshooting).

