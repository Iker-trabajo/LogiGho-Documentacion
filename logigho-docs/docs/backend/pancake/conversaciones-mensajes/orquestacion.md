## Autor: Iker Acevedo
Fecha creación: 2026-08-04

Estado: producción

# Orquestación — Step Functions + EventBridge

`Step_Conversaciones_Pancake` coordina 3 lambdas (2 propias + 1 reusada) y **se llama a sí mismo** para drenar toda la cola de pendientes en 1 sola ejecución, sin importar cuántas conversaciones haya.

---

## Diagrama de estados

Estados reales del Step Function `Step_Conversaciones_Pancake`, tal cual quedaron desplegados en AWS (mismos nombres que la definición ASL de abajo):

```mermaid
--8<-- "backend/pancake/conversaciones-mensajes/diagramas/orquestacion-estados.mmd"
```

> Fuente de este diagrama: **[carpeta de diagramas](diagramas/README.md)**.

---

## Por qué el self-loop (y no un `Map`)

El proceso de Estadísticas usa un `Map` para abanicar trabajo (1 iteración por página). Acá **no se pudo** hacer lo mismo para mensajes: la lista de conversaciones pendientes puede tener **millones de filas**, y el estado de un Step Function tiene un techo duro de **256 KB** — pasar esa lista completa como input de un `Map` revienta con `States.DataLimitExceeded` (pasó en producción, ver historial de cambios de la [Visión general](overview.md)).

La solución: `ApiLambdaSincronizarMensajes` **nunca recibe la lista por el Step Function** — la consulta ella misma contra MySQL en cada invocación (`LIMIT` chico, `LIMITE_CONVERSACIONES_PENDIENTES`), procesa lo que le alcance en sus 15 minutos, y devuelve solo un **conteo**. El `Choice` decide si hay que volver a invocarla mirando ese conteo, no la lista.

---

## Conceptos clave

### `MapSincronizarConversaciones` — 1 invocación por página

`ItemsPath: "$"` — el input completo del Step Function (la lista de páginas activas) se reparte, 1 página por iteración. `MaxConcurrency: 8` limita cuántas páginas se sincronizan a la vez (cada una hace su propio barrido secuencial de conversaciones vía watermark).

### `Catch` con `IgnorarErrorPagina` — el radio de impacto es 1 página

Si una página falla (token vencido sin poder regenerar, error de red persistente, etc.), esa iteración cae a un `Pass` y **el resto de páginas sigue**. Un problema en 1 página no frena la sincronización de las demás.

### El self-loop — `SincronizarMensajesPendientes` → `QuedanPendientes` → vuelve a sí misma

Cada vuelta del loop:
1. Invoca `ApiLambdaSincronizarMensajes`, que trae un lote (`LIMITE_CONVERSACIONES_PENDIENTES`, hoy 800) de conversaciones pendientes de **cualquier página**, las procesa en paralelo, y corta si se queda sin tiempo (margen de seguridad de 30s antes del timeout de Lambda).
2. Devuelve `{ ConversacionesProcesadas, ConversacionesConError, TotalMensajesSincronizados, SeCortoPorTiempo }`.
3. `QuedanPendientes` mira `ConversacionesProcesadas`: si es `> 0`, vuelve a invocar (todavía puede haber más pendientes); si es `0`, no queda nada por procesar y termina.

Con esto, **1 sola ejecución del Step Function drena toda la cola**, sin importar si son 800 o 400,000 conversaciones — solo tarda más vueltas del loop.

---

## Definición (ASL)

```json
{
  "Comment": "Pipeline sincronizacion conversaciones y mensajes Pancake",
  "StartAt": "ObtenerPaginasActivas",
  "States": {
    "ObtenerPaginasActivas": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:376313750428:function:ApiLambdaObtenerPaginasActivas",
      "Retry": [
        { "ErrorEquals": ["Lambda.ServiceException", "Lambda.AWSLambdaException", "Lambda.SdkClientException", "Lambda.TooManyRequestsException"],
          "IntervalSeconds": 2, "MaxAttempts": 3, "BackoffRate": 2 }
      ],
      "Next": "MapSincronizarConversaciones"
    },
    "MapSincronizarConversaciones": {
      "Type": "Map",
      "MaxConcurrency": 8,
      "ItemsPath": "$",
      "ItemSelector": { "PageId.$": "$$.Map.Item.Value.page_id" },
      "ItemProcessor": {
        "ProcessorConfig": { "Mode": "INLINE" },
        "StartAt": "SincronizarConversaciones",
        "States": {
          "SincronizarConversaciones": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:us-east-1:376313750428:function:ApiLambdaSincronizarConversacionesPancake",
            "Retry": [
              { "ErrorEquals": ["Lambda.ServiceException", "Lambda.AWSLambdaException", "Lambda.SdkClientException", "Lambda.TooManyRequestsException"],
                "IntervalSeconds": 3, "MaxAttempts": 2, "BackoffRate": 2 }
            ],
            "Catch": [ { "ErrorEquals": ["States.ALL"], "ResultPath": "$.error", "Next": "IgnorarErrorPagina" } ],
            "End": true
          },
          "IgnorarErrorPagina": { "Type": "Pass", "End": true }
        }
      },
      "Next": "SincronizarMensajesPendientes"
    },
    "SincronizarMensajesPendientes": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:376313750428:function:ApiLambdaSincronizarMensajes",
      "Retry": [
        { "ErrorEquals": ["Lambda.ServiceException", "Lambda.AWSLambdaException", "Lambda.SdkClientException", "Lambda.TooManyRequestsException"],
          "IntervalSeconds": 3, "MaxAttempts": 2, "BackoffRate": 2 }
      ],
      "Next": "QuedanPendientes"
    },
    "QuedanPendientes": {
      "Type": "Choice",
      "Choices": [
        { "Variable": "$.ConversacionesProcesadas", "NumericGreaterThan": 0, "Next": "SincronizarMensajesPendientes" }
      ],
      "Default": "FinSincronizacion"
    },
    "FinSincronizacion": { "Type": "Pass", "End": true }
  }
}
```

---

## Agendamiento — EventBridge Scheduler

| Parámetro | Valor |
|---|---|
| Occurrence | Recurring schedule |
| Schedule type | Cron-based |
| Cron | `cron(0 9,12,15,18 * * ? *)` |
| Timezone | `America/Bogota` |
| Flexible time window | Off |
| Target | AWS Step Functions → `StartExecution` → `Step_Conversaciones_Pancake` |
| Input | `{}` (la 1ª lambda no necesita nada propio) |
| Retry policy | `Retry` habilitado — Maximum age of event `1 hora`, Retry attempts `3` (no `24h`/`185` default: si `StartExecution` falla, reintentar por 24h no tiene sentido cuando la próxima corrida ya llega en 3h) |
| Execution role | Autogenerado por el wizard (`Amazon_EventBridge_Scheduler_SFN_...`) |

Corre 4 veces al día: **9am, 12pm, 3pm, 6pm** (hora Colombia).

> ¿Por qué no se solapan las corridas de las 12pm y 3pm si una tarda mucho? Ver [Deuda técnica](overview.md#deuda-tecnica-pendientes) en la Visión general — hoy el riesgo es bajo porque las corridas terminan en minutos, pero no hay una protección explícita todavía.

---

## Configuración del pipeline

| Parámetro | Valor |
|---|---|
| Nombre | `Step_Conversaciones_Pancake` |
| Tipo | Standard |
| Cuenta AWS | `376313750428` |
| Región | `us-east-1` |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-08-04 | Iker Acevedo | Creación del Step Function con `Map` de conversaciones + invocación directa de mensajes. |
| 2026-08-05 | Iker Acevedo | Rediseño de `SincronizarMensajesPendientes`: pasa de recibir la lista por el estado (rompía con `States.DataLimitExceeded`) a autoconsultar MySQL y devolver solo un conteo. Se agrega el self-loop vía `Choice`. |
| 2026-08-07 | Iker Acevedo | EventBridge Scheduler agendado cada 3h (9am–6pm), con `Retry policy` acotada (1h / 3 intentos). |

---

## Observaciones

- El `Retry` de cada `Task` cubre errores **transitorios de Lambda** (throttling, cold start del servicio) — no cubre errores de negocio (ej. token de Pancake vencido), esos se manejan **dentro** de cada lambda.
- `MaxConcurrency: 8` en `MapSincronizarConversaciones` es deliberadamente bajo: cada página ya pagina internamente su propio historial de conversaciones, no hace falta abanicar mucho para saturar el trabajo disponible.
