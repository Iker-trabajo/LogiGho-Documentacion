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
| `webhook` | `CallbacksRecibidos`, `CallbacksProcesados`, `MotivosNuevos`, `CallbacksIgnorados`, `CallbacksDesconocidos`, `CallbacksInvalidos`, `WebhookRechazado`, `CallbackErrorInterno`, `CallbacksConErrorProveedor` |

## Alarmas de CloudWatch (Terraform, `modules/novedades-inter/alarmas.tf`) → SNS → correo

`ws-inter-no-disponible`, `corrida-cortada`, `api-key-rechazada`, `guias-sin-callback`, `webhook-error-interno` (el webhook respondió 500), `webhook-error-proveedor` (≥3 callbacks de error del proveedor en 5 min),
`webhook-rechazado` (≥3 en 5 min), `lambda-errores`, `lambda-throttles`, `lambda-cerca-del-timeout` (90 %) y,
solo en producción, **`corrida-ausente`**: 9 h sin ninguna corrida (la Step está apagada, fallando o sin permiso para
invocar la lambda — la Step oculta ese error con su `Catch`).

## Dashboard

`Novedades-Inter-<entorno>` (CloudWatch): envíos, salud del proveedor (debe estar en 0), callbacks, mantenimiento,
lambda (invocaciones/errores/duración) y estado de todas las alarmas.

> No hay alarmas ni gráficos de API Gateway: el REST es compartido por toda la plataforma y sus métricas por stage mezclan
> todas las rutas. El webhook se vigila con las métricas que publica la propia lambda. Un 403 que API Gateway da por API key
> inválida no llega a la lambda; se detecta porque no llegan callbacks (`guias-sin-callback`).


---

# Historia 027: Distancias y Reportes

Las dos funciones nuevas comparten el namespace `Logigho/NovedadesInter`, el tema SNS `Novedades-Inter-Alarmas-<entorno>` y las etiquetas de la
función original. Observado: `modules/novedades-inter/funciones-distancias-reportes.tf` y `alarmas.tf` (Terraform `main`, PR 621).

## Alertas de Slack (las manda la propia función)

| Función | Alerta | Significa | Qué hacer |
| ------- | ------ | --------- | --------- |
| Distancias | **Google Maps está fallando: se cortó la corrida de distancias** | `FALLAS_SEGUIDAS_PARA_CORTAR` fallas seguidas de Google o de escrituras | Revisar la llave, las cuotas y el estado de Google; lo pendiente se retoma en la siguiente corrida |
| Reportes | **No se pudieron generar los reportes de novedades Inter (corrida …)** | Falló la lectura, el Excel o la subida a Slack | Revisar los logs; se reintenta en el próximo reporte (la corrida queda `Fallida`) |
| Reportes | **Los reportes se entregaron, pero no se pudo registrar la corrida** | Los archivos llegaron, pero no se escribió `ReportesNovedadesInter` | Las «Nuevas» del próximo reporte pueden repetirse; revisar Mongo |

Los avisos de Reportes van al canal de reportes (`SLACK_CANAL_REPORTES`) con el bot; la alerta de Distancias, al webhook de alertas.

## Métricas (namespace `Logigho/NovedadesInter`, dimensiones `Entorno` y `Operacion`)

| Operación | Métricas |
| --------- | -------- |
| `distancias` | `GuiasSeleccionadas`, `GuiasProcesadas`, `GuiasConError`, `Geocodificaciones`, `Rutas`, `LlamadasGoogle`, `PendientesPorTope`, `PendientesPorTiempo`, `CandadoOcupado`, `CorridaCortada`, `CorridaInterrumpida` |
| `reportes` | `ReportesExitosos`, `ReportesFallidos`, `ReportesNoRegistrados`, `FilasReporte1Nuevas`, `FilasReporte1Acumulado`, `FilasReporte2Nuevas`, `FilasReporte2Acumulado`, `ComandoSlackFirmaRechazada`, `ComandoSlackUsuarioNoPermitido` |

## Alarmas de CloudWatch nuevas (4 por entorno)

Nombres tal como están en Terraform (`<entorno>` = `preprod` o `prod`); todas envían a SNS al pasar a alarma y al volver a OK, con `treat_missing_data = notBreaching` y periodo de 300 s.

| Alarma | Métrica | Umbral |
| ------ | ------- | ------ |
| `Novedades-Inter-<entorno>-distancias-errores` | `AWS/Lambda` `Errors` de `ApiLambdaNovedadesInterDistancias` (Sum) | mayor que 0 |
| `Novedades-Inter-<entorno>-reportes-errores` | `AWS/Lambda` `Errors` de `ApiLambdaNovedadesInterReportes` (Sum) | mayor que 0 |
| `Novedades-Inter-<entorno>-distancias-cerca-del-timeout` | `AWS/Lambda` `Duration` (Maximum) | mayor que 270 000 ms (90 % de 300 s) |
| `Novedades-Inter-<entorno>-reportes-cerca-del-timeout` | `AWS/Lambda` `Duration` (Maximum) | mayor que 270 000 ms (90 % de 300 s) |

- Con las 10 existentes de la función original, PreProd tiene **14 alarmas**, todas en estado OK el 2026-11-06 (observado en la validación). En producción
  se suma `corrida-ausente`; **pendiente** hasta que haya despliegue.
- **Brecha conocida:** `CorridaCortada` de Distancias se emite como métrica y avisa por Slack, pero la alarma `corrida-cortada` existente mira
  `Operacion = solicitar`; no hay alarma de CloudWatch para la de Distancias. Tampoco hay alarma para `ReportesFallidos` ni para `ComandoSlackFirmaRechazada`.
  Se detectan por el aviso de Slack y por las métricas.
- **Dashboard:** `Novedades-Inter-<entorno>` no se extendió con gráficos de las funciones nuevas (observado: sin referencias en `dashboard.tf`). Las métricas se ven en CloudWatch.
- El Scheduler de reportes no tiene alarma propia: si no corre, se nota porque no llega el reporte (hipótesis; **pendiente** decidir si se agrega una alarma de ausencia).
