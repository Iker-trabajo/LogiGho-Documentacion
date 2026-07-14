## Autor: Iker Acevedo
Fecha creacion: 2026-07-13

Estado: produccion

# Orquestación — Step Functions + EventBridge

Las 5 lambdas no se llaman entre sí: las coordina un **Step Function** (`PipelineEstadisticasPancake`), y **EventBridge Scheduler** lo dispara 4 veces al día. Este documento explica cómo encaja todo.

---

## Diagrama de estados

```
Start
  │
  ▼
CalcularVentana  ──► ResultPath $.ventana
  │
  ▼
ObtenerCuentas   ──► ResultPath $.cuentas
  │
  ▼
MapPorCuentaMadre  (ItemsPath $.cuentas · MaxConcurrency 3)
  │   └── ListarPaginas ──(Catch)──► ErrorCuenta
  ▼
ObtenerPaginasActivas  ──► ResultPath $.paginasActivas
  │
  ▼
MapPorPagina  (ItemsPath $.paginasActivas · MaxConcurrency 5 · ItemSelector)
  │   └── ObtenerEstadisticas ──(Retry)──(Catch)──► ErrorPagina
  ▼
End
```

---

## Conceptos clave

### `ResultPath` — acumular sin pisar

En vez de reemplazar todo el estado con la salida de cada lambda, se guarda en un **sub-campo** y se conserva lo demás. Así el estado va acumulando:

```
{ slot_id, tipo_calculo }                      ← input original (del cron)
  + $.ventana         { since, until, ... }    ← de CalcularVentana
  + $.cuentas         [ ... ]                  ← de ObtenerCuentas
  + $.paginasActivas  [ ... ]                  ← de ObtenerPaginasActivas
```

### `Map` — abanicar (fan-out)

- **1er Map** (`MapPorCuentaMadre`): una iteración por cuenta madre → `ListarPaginas`. `MaxConcurrency 3` (pocas cuentas).
- **2º Map** (`MapPorPagina`): una iteración por página → `ObtenerEstadisticas`. `MaxConcurrency 5`.

### `ItemSelector` — inyectar la ventana a cada página

El 2º Map **combina** cada página con el contexto de la ventana, para que `ObtenerEstadisticas` reciba todo lo que necesita en un solo objeto:

```json
"ItemSelector": {
  "page_id.$": "$$.Map.Item.Value.page_id",
  "page_access_token.$": "$$.Map.Item.Value.page_access_token",
  "timezone.$": "$$.Map.Item.Value.timezone",
  "since.$": "$.ventana.since",
  "until.$": "$.ventana.until",
  "fecha_reporte.$": "$.ventana.fecha_reporte",
  "tipo.$": "$.ventana.tipo",
  "slot_id.$": "$.ventana.slot_id"
}
```

### `Retry` + `Catch` — resiliencia declarativa

- **`Retry`** (antes que `Catch`): reintenta los errores **transitorios** de Lambda con backoff exponencial (2s → 4s → 8s...). Cubre el throttling (`Lambda.TooManyRequestsException`) cuando el Map abanica muchas invocaciones.
- **`Catch`**: si se agotan los reintentos, la iteración cae a un estado `Pass` (`ErrorCuenta` / `ErrorPagina`) y **el Map continúa** con las demás. El radio de impacto de un fallo es 1 elemento, no todo el pipeline.

---

## Definición completa (ASL)

```json
{
  "Comment": "PipelineEstadisticasPancake",
  "StartAt": "CalcularVentana",
  "States": {
    "CalcularVentana": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:376313750428:function:ApiLambdaCalcularVentanaTiempo",
      "ResultPath": "$.ventana",
      "Next": "ObtenerCuentas"
    },
    "ObtenerCuentas": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:376313750428:function:ApiLambdaObtenerCuentasPrincipales",
      "ResultPath": "$.cuentas",
      "Next": "MapPorCuentaMadre"
    },
    "MapPorCuentaMadre": {
      "Type": "Map",
      "ItemsPath": "$.cuentas",
      "MaxConcurrency": 3,
      "Iterator": {
        "StartAt": "ListarPaginas",
        "States": {
          "ListarPaginas": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:us-east-1:376313750428:function:ApiLambdaListarPaginasPancake",
            "Retry": [
              { "ErrorEquals": ["Lambda.TooManyRequestsException", "Lambda.ServiceException", "Lambda.SdkClientException", "Lambda.AWSLambdaException"],
                "IntervalSeconds": 2, "MaxAttempts": 6, "BackoffRate": 2 }
            ],
            "End": true,
            "Catch": [ { "ErrorEquals": ["States.ALL"], "Next": "ErrorCuenta" } ]
          },
          "ErrorCuenta": { "Type": "Pass", "End": true }
        }
      },
      "ResultPath": "$.resultadoPaginas",
      "Next": "ObtenerPaginasActivas"
    },
    "ObtenerPaginasActivas": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:376313750428:function:ApiLambdaObtenerPaginasActivas",
      "ResultPath": "$.paginasActivas",
      "Next": "MapPorPagina"
    },
    "MapPorPagina": {
      "Type": "Map",
      "ItemsPath": "$.paginasActivas",
      "MaxConcurrency": 5,
      "ItemSelector": {
        "page_id.$": "$$.Map.Item.Value.page_id",
        "page_access_token.$": "$$.Map.Item.Value.page_access_token",
        "timezone.$": "$$.Map.Item.Value.timezone",
        "since.$": "$.ventana.since",
        "until.$": "$.ventana.until",
        "fecha_reporte.$": "$.ventana.fecha_reporte",
        "tipo.$": "$.ventana.tipo",
        "slot_id.$": "$.ventana.slot_id"
      },
      "Iterator": {
        "StartAt": "ObtenerEstadisticas",
        "States": {
          "ObtenerEstadisticas": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:us-east-1:376313750428:function:ApiLambdaObtenerEstadisticas",
            "Retry": [
              { "ErrorEquals": ["Lambda.TooManyRequestsException", "Lambda.ServiceException", "Lambda.SdkClientException", "Lambda.AWSLambdaException"],
                "IntervalSeconds": 2, "MaxAttempts": 6, "BackoffRate": 2 }
            ],
            "End": true,
            "Catch": [ { "ErrorEquals": ["States.ALL"], "Next": "ErrorPagina" } ]
          },
          "ErrorPagina": { "Type": "Pass", "End": true }
        }
      },
      "End": true
    }
  }
}
```

> **Permisos:** al crear la máquina de estados, se dejó que la consola generara un rol nuevo — que incluye automáticamente `lambda:InvokeFunction` sobre las 5 funciones.

---

## Agendamiento — EventBridge Scheduler

4 *schedules* recurrentes, con zona horaria **`America/Bogota`** (así se usan horas locales sin convertir a UTC). Cada uno dispara el Step Function con un input distinto.

| Schedule | Hora Colombia | Cron | Input |
| -------- | ------------- | ---- | ----- |
| `pancake-slot1-7am` | 7:00 am | `cron(0 7 * * ? *)` | `{ "slot_id": "1", "tipo_calculo": "cierre_dia_anterior" }` |
| `pancake-slot2-9am` | 9:00 am | `cron(0 9 * * ? *)` | `{ "slot_id": "2", "tipo_calculo": "intradia_actual" }` |
| `pancake-slot3-2pm` | 2:00 pm | `cron(0 14 * * ? *)` | `{ "slot_id": "3", "tipo_calculo": "intradia_actual" }` |
| `pancake-slot4-5pm` | 5:00 pm | `cron(0 17 * * ? *)` | `{ "slot_id": "4", "tipo_calculo": "intradia_actual" }` |

> Formato cron de AWS: `cron(min hora dia-mes mes dia-semana año)`. El `?` = "sin valor específico" en día-mes/día-semana (no se pueden fijar ambos).

- **Target:** AWS Step Functions → `StartExecution` → `PipelineEstadisticasPancake`.
- **Rol:** los 4 schedules comparten un rol con permiso `states:StartExecution`.

> Para **agregar** o **cambiar** una franja, ver **[Operación → Agregar nuevas ejecuciones](operacion.md#agregar-nueva-ejecucion)**.

---

## Configuración del pipeline

| Parámetro | Valor |
| --------- | ----- |
| Nombre | `PipelineEstadisticasPancake` |
| Tipo | Standard |
| Cuenta AWS | `376313750428` |
| Región | `us-east-1` |
| Rol de ejecución | rol generado por la consola (con `lambda:InvokeFunction`) |

---

## Historial de cambios

| Fecha | Autor | Cambio |
| ----- | ----- | ------ |
| 2026-07-13 | Iker Acevedo | Creación del Step Function (2 Map, ResultPath, ItemSelector, Catch). |
| 2026-07-13 | Iker Acevedo | Agregado `Retry` con backoff ante throttling de Lambda; `MaxConcurrency` del 2º Map ajustado de 20 a 5. |
| 2026-07-13 | Iker Acevedo | 4 schedules de EventBridge (zona `America/Bogota`) apuntando al Step Function. |

---

## Observaciones

- `CalcularVentana` y `ObtenerCuentas` van **secuenciales** por simplicidad (ambas son muy rápidas). Se podrían paralelizar con un `Parallel`, pero la ganancia es marginal frente a la complejidad.
- El `Retry` va **siempre antes** que el `Catch`: primero se reintenta (transitorios), y solo si se agota se captura el fallo.
