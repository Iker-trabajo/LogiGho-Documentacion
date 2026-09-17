## Autor: Iker Acevedo
Fecha creación: 2026-09-16

Estado: preprod

# Orquestación en PreProd — dominio `financiero`

Para poder disparar liquidaciones de prueba sin tocar producción, se construyó un dominio AWS nuevo en preprod (cuenta `376313750428`) siguiendo la receta estándar del repo (`.preprod-hub/06_RECETA_NUEVO_DOMINIO.md`): IaC, permisos y pipeline propios, deliberadamente mínimos.

---

## Step Function mínimo: un solo paso

Diseño deliberadamente simple vs Pancake (que necesitó `Map`, loop, `Choice` desde día uno). Objetivo: invocar Lambda directo, ver resultado.

**Sin complejidad extra porque:**
- Motor Dinámico: Mongo directo (sin dependencias auth).
- Inventario: se salta si token vacío (ver `Function.cs`).
- Ni Cognito ni validación externa necesarias.

**Evolución futura:** agregar Task por Task (Token, backup S3, notificación) — mismo patrón que `procesos-automaticos` usó al crecer.

```json
{
  "Comment": "Liquida en PreProd las guias pendientes (legacy + motor Dinamico TCC segun ConfiguracionLiquidacionTransportadora). Se inicia a mano.",
  "StartAt": "Lambda Liquidaciones",
  "States": {
    "Lambda Liquidaciones": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "${ApiLambdaLiquidacionesLogighoAOT.Arn}",
        "Payload": {
          "body.$": "States.JsonToString($)",
          "headers": {}
        }
      },
      "Retry": [
        { "ErrorEquals": ["Lambda.ServiceException", "Lambda.AWSLambdaException", "Lambda.SdkClientException", "Lambda.TooManyRequestsException"],
          "IntervalSeconds": 5, "MaxAttempts": 2, "BackoffRate": 2 }
      ],
      "End": true
    }
  }
}
```

`"body.$": "States.JsonToString($)"` pasa el `Input` de la ejecución **tal cual** al `body` de la Lambda — lo que se escriba en "Start execution" es exactamente el JSON que recibe el handler. Así se puede liquidar todas las guías pendientes (`{}`) o una lista puntual (`{"Guias":["700000001"]}`), sin dos Step Functions distintos.

---

## Infraestructura como código

`.preprod-hub/infra/templates/financiero-infra-preprod.yaml` — todo nuevo, sin `resource import`:

| Recurso | Detalle |
|---|---|
| `AWS::Lambda::Function` | `ApiLambdaLiquidacionesLogighoAOT`, `dotnet8`, 2048 MB, timeout 900s |
| `AWS::IAM::Role` | `StepFunctions-Financiero-PreProd` — solo puede invocar esta única Lambda |
| `AWS::StepFunctions::StateMachine` | `Financiero-PreProd` |

### Variables de entorno de la Lambda

| Variable | Para qué |
|---|---|
| `CADENA_CONEXION`, `ID_CLIENTE`, `USER_AUTH`, `TOKEN_AUTH`, `SUCURSAL_GENERICA` | Cifradas AES — 4 de las 5 solo se descifran en el constructor legacy pero nunca se vuelven a leer en el resto del código (confirmado por grep); solo `CADENA_CONEXION` es funcionalmente necesaria hoy |
| `DATABASE_NAME` | `LogighoDB` |
| `MODIFICACION_ETIQUETA` | `"false"` |
| `URL_SERVICIO_AWS` | `BaseAddress` del `HttpClient` que usa `Generico.consumoGenerico` — necesaria en cuanto se mezcla un pedido **no-TCC** en la misma ejecución (ver [Observaciones](ApiLambdaLiquidacionesLogighoAOT.md#observaciones)) |

### Permisos en la cuenta

`GitHubActionsDeployRole` (definido en `cuenta-infra-preprod.yaml`) necesitó 3 statements nuevos, con el mismo patrón que ya existía para `procesos-automaticos` pero acotados a `Financiero-*`:

- `ManejarStepFunctionsFinanciero` — crear/actualizar/borrar el Step Function, acotado a `stateMachine:Financiero-*`.
- `ManejarRolesDeFinanciero` — crear/gestionar el rol `StepFunctions-Financiero-PreProd`.
- `EntregarRolesDeFinanciero` — `iam:PassRole` sobre ese mismo rol.

Sin esto, el primer `aws cloudformation deploy` del stack `financiero-preprod` habría fallado por permisos insuficientes del rol de despliegue — el statement existente de `procesos-automaticos` estaba acotado solo a `stateMachine:ProcesosAutomaticos-*`.

---

## Pipeline (GitHub Actions)

`.github/workflows/deploy-financiero-preprod.yml`, mismo patrón que el resto de dominios del repo: un job de `test` (117 tests, Mongo real como `services:` container) que debe pasar antes de que el job de `deploy` corra (`needs: test`), notificaciones de Slack en los 4 puntos estándar (inicio, resultado de tests, éxito, fallo — ver `.preprod-hub/07_ESTANDAR_SLACK_NOTIFICACIONES.md`), y despliegue con `aws cloudformation deploy --stack-name financiero-preprod` pasando los 6 parámetros cifrados + la key de S3 del paquete.

Se dispara solo cuando cambia algo dentro de `ApiLambdaLiquidacionesLogighoAOT/**`, `.Tests/**`, o el propio template.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-09-16 | Iker Acevedo | Dominio `financiero` construido y desplegado por primera vez: template, permisos, pipeline. Stack confirmado `CREATE_COMPLETE`. |
| 2026-09-16 | Iker Acevedo | `URL_SERVICIO_AWS` agregada tras el primer error real en preprod (pedidos no-TCC en la misma ejecución que TCC). |
