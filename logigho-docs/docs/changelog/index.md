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

## [2026-09-16] — Liquidaciones: motor Dinámico (TCC) conviviendo con el motor Legacy

### Nuevas funcionalidades

- **Motor Dinámico de liquidaciones**, dentro de la misma Lambda `ApiLambdaLiquidacionesLogighoAOT` (patrón Strangler Fig, ver ADR-001): dominio limpio (modelos, calculadoras, políticas de peso, validadores), configurable por transportadora vía `ConfiguracionLiquidacionTransportadora` (`Dinamica`/`Legacy`/`Sombra`/`Deshabilitado`). Liquida hoy TCC; el motor Legacy sigue intacto para Interrapidísimo, Envía y Servientrega.
- **Módulo frontend "Costos Transportadora"** (`director-de-operaciones`): Hub de transportadoras + panel de 5 pestañas (Configuración General, Trayectos, Tarifas por Tienda, Simulador de Liquidación, Auditoría de Excepciones) para configurar el motor Dinámico sin desplegar código.
- **Dominio AWS `financiero`** en PreProd: Lambda + Step Function `Financiero-PreProd`, para ejecutar liquidaciones de prueba (todas las pendientes, o una lista puntual de guías) sin tocar producción.

### Correcciones

- `TarifaTransportadoraTienda.Trayectos` e `IncrementosPeso` cambiados de `IReadOnlyList<T>` a `List<T>` — Newtonsoft no puede poblar una `ReadOnlyCollection` al deserializar directo desde Mongo (`JsonSerializationException` real visto en CloudWatch). Se agregaron 3 tests de regresión que deserializan JSON con forma real, cerrando el hueco de cobertura que dejó pasar el bug.
- `URL_SERVICIO_AWS` agregada al template de `financiero-infra-preprod.yaml` — faltaba para que el camino legacy no-TCC (`Generico.consumoGenerico`) pudiera actualizar inventario cuando se mezclan pedidos legacy y TCC en la misma ejecución.

### Documentación

- Nueva sección **Backend → Lambdas .NET → LogiGho → ApiLambdaLiquidacionesLogighoAOT**: visión general con diagrama de flujo estilo BPMN, motor Legacy, motor Dinámico completo (orquestador, modelos, calculadoras, políticas de peso, validación, infraestructura), despliegue y operación en PreProd, ADR-001.
- Nueva sección **Frontend → Director de Operaciones → Costos Transportadora**: componente principal, las 5 pestañas, modelos, servicio de estado, ADR-001.
- Nueva página **Frontend → Analytics → Liquidaciones (Power BI)**.

---

## [2026-09-02] — DevolucionesMasivo: confiabilidad de Inter (token real, reintentos, fallback cruzado)

### Correcciones

- **Token de Inter vencía en 20 minutos según el código, en 30 segundos según Inter real** (confirmado decodificando el JWT que devuelve `GenerarTokenTemporal`) — causaba 401 masivos en cargas de más de 30s de duración. `_tokenVence` bajado a 20s.
- Fallos HTTP de `ClienteInter` (rastreo y estados) no dejaban ningún rastro en logs cuando Inter respondía distinto de 2xx — ahora se loguea `statusCode` y cuerpo de la respuesta en cada intento fallido.
- Sin reintentos ante `429`/`5xx` de Inter: ahora hasta 4 intentos con backoff exponencial + jitter, respetando `Retry-After`.
- Tráfico sostenido sin pausas entre tandas (`SemaphoreSlim` sin cortes) podía seguir gatillando 429 con lotes de 200+ guías — ahora hay pausa configurable entre tanda y tanda (`PAUSA_ENTRE_PETICIONES_INTER_MS`).
- Guías rechazadas por la transportadora asignada (`ErrorConsultaExterna`, ej. 400 explícito) o sin ningún formato reconocido (`GuiaNoExisteEnSistema`) no se reintentaban contra otra transportadora — solo cubría `SinGuiaOriginalEnRespuesta`. Ahora los 3 motivos disparan el fallback cruzado, con Inter primero por volumen.

### Cambios

- `CONCURRENCIA_INTER` por defecto bajada de 10 a 5 en código y en ambos entornos (prod/preprod).

### Documentación

- [ADR-005](../backend/lambdas-dotnet/aplicacion/Logigho/ApiLambdaDevolucionesMasivo/adr/ADR-005-confiabilidad-inter.md) con el diagnóstico completo. Actualizados `clientes-transportadora.md`, `worker-handler.md` y la tabla de variables de entorno del módulo.

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

