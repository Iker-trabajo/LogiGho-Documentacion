## Autor: Iker Acevedo

Fecha creacion: 2026-10-18  
Estado: desarrollo

---

# Observabilidad y monitoreo

> Estándar desde este dominio: **todo servicio nuevo** lleva logs estructurados, métricas, alarmas, dashboard y alerta
> inmediata por Slack cuando falla una dependencia externa.

## Qué se avisa y a quién

| Canal | Contenido | Quién lo configura |
| ----- | --------- | ------------------ |
| Slack `#logigho-alertas-prod` | Solo lo que exige acción ya (ws-inter caído, API key rechazada, callbacks que no llegan, error no controlado) | Webhook en `SLACK_WEBHOOK_ALERTAS` (variable group `novedades-inter-prod-secretos`) |
| Slack `#logigho-alertas-preprod` | Lo mismo en PreProd (opcional) | Mismo mecanismo; sin webhook solo se escribe el log |
| Correo (SNS) | **Respaldo**: alarmas de CloudWatch por si la lambda se cae y no alcanza a avisar | Terraform (`correos_alarma`); cada correo debe **confirmar la suscripción** |
| Slack de despliegues | Inicio/resultado de los pipelines | Ya existente (`slack-notificaciones`) |

Los canales se separan por **entorno y urgencia**, no por servicio.

## Alertas de Slack (las manda la propia lambda)

Máximo una por tipo y corrida. Cada mensaje trae entorno, hora de Colombia y un botón "Ver logs" al grupo
`/aws/lambda/ApiLambdaNovedadesInter`.

| Alerta | Significa | Qué hacer |
| ------ | --------- | --------- |
| **ws-inter NO responde** | `/health` falló o todas las solicitudes fallaron | Llamar al proveedor |
| **ws-inter rechazó nuestra API key** | 401/403 | Pedir/renovar la clave con el proveedor (ver runbook) |
| **ws-inter está fallando: se cortó la corrida** | 5 fallas seguidas | Llamar al proveedor |
| **ws-inter recibe las guías pero no devuelve el callback** | 3 envíos sin respuesta | Verificar con el proveedor la URL y la clave del webhook |
| **Falló la corrida de novedades Inter** | Excepción (Mongo, programación) | Revisar logs |

## Logs

Una línea JSON por evento (`Infraestructura/Utilidades/Log.cs`) con `ts`, `modulo`, `requestId`, `ruta`
(`solicitar`|`webhook`), `nivel`, `etapa` y datos (`guia`, `ref`, …). Ejemplo en CloudWatch Logs Insights:

```
fields @timestamp, etapa, guia, error | filter etapa = "solicitud-fallida" | sort @timestamp desc
```

## Métricas (namespace `Logigho/NovedadesInter`, dimensiones `Entorno` y `Operacion`)

Se publican en formato EMF (una línea por invocación, sin llamadas de red).

| Operación | Métricas |
| --------- | -------- |
| `solicitar` | `GuiasElegibles`, `SolicitudesEnviadas`, `SolicitudesFallidas`, `GuiasSaltadas`, `PendientesPorTope`, `SeguimientosCerrados`, `MotivosReproyectados`, `GuiasMarcadasSinRespuesta`, `WsInterNoDisponible`, `CorridaCortada`, `CredencialInvalida`, `CorridaInterrumpida` |
| `webhook` | `CallbacksRecibidos`, `CallbacksProcesados`, `MotivosNuevos`, `CallbacksIgnorados`, `CallbacksDesconocidos`, `CallbacksInvalidos`, `WebhookRechazado`, `CallbackErrorInterno` |

## Alarmas de CloudWatch (Terraform, `modules/novedades-inter/alarmas.tf`) → SNS → correo

`ws-inter-no-disponible`, `corrida-cortada`, `api-key-rechazada`, `guias-sin-callback`, `webhook-error-interno` (el webhook respondió 500),
`webhook-rechazado` (≥3 en 5 min), `lambda-errores`, `lambda-throttles`, `lambda-cerca-del-timeout` (90 %) y,
solo en producción, **`corrida-ausente`**: 9 h sin ninguna corrida (la Step está apagada, fallando o sin permiso para
invocar la lambda — la Step oculta ese error con su `Catch`).

## Dashboard

`Novedades-Inter-<entorno>` (CloudWatch): envíos, salud del proveedor (debe estar en 0), callbacks, mantenimiento,
lambda (invocaciones/errores/duración) y estado de todas las alarmas.

> No hay alarmas ni gráficos de API Gateway: el REST es compartido por toda la plataforma y sus métricas por stage mezclan
> todas las rutas. El webhook se vigila con las métricas que publica la propia lambda. Un 403 que API Gateway da por API key
> inválida no llega a la lambda; se detecta porque no llegan callbacks (`guias-sin-callback`).
