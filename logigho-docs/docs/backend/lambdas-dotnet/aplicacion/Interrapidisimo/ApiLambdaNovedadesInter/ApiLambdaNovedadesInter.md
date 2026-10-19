## Autor: Iker Acevedo

Fecha creacion: 2026-10-18  
Estado: desarrollo

---

## Lambda: ApiLambdaNovedadesInter

**Trigger:** Step Function (`Step_Procesos_Logigho`) + API Gateway REST (webhook)  
**AOT:** No (runtime administrado `dotnet10`)

---

## ¿Qué hace?

Descubre **por qué** una guía de Interrapidísimo quedó en novedad. La API oficial de Inter solo informa el estado; el
proveedor externo **ws-inter** (scraping del "Flujo Guía" de Inter) entrega además el **motivo** (por ejemplo
`REHUSADO / SE NEGO A RECIBIR NOTIFICACION`) y la **ubicación (geo)** donde el mensajero lo registró.

La lambda envía a ws-inter las guías en novedad, recibe el resultado por un webhook, lo guarda en la colección
`SeguimientoNovedadesInter` y deja en `PedidosInter` un único campo legible: `MotivoNovedadInter`.

```
MotivoNovedadInter = "REHUSADO / SE NEGO A RECIBIR NOTIFICACION - https://www.google.com/maps?q=10.868765635,-74.78554886"
```

Es **solo para mostrar**: nunca se debe parsear. El detalle (fecha, ciudad, mensajero, geo, historial) vive en
`SeguimientoNovedadesInter`, que se busca por número de guía (`Numeropreenvio`).

---

## Accionador

Una sola Lambda con **dos rutas**, elegidas por el `path` del evento (`Function.cs`):

| Método | Ruta | Quién la llama | Auth |
| ------ | ---- | -------------- | ---- |
| `POST` | `/novedades-inter/solicitar` | La Step Function, justo después de `Lambda Pedidos Inter` (no existe en API Gateway) | IAM (rol de la Step) |
| `POST` | `/webhooks/ws-inter/flujo-guia` | ws-inter, por API Gateway REST | Header `x-api-key` |

Detalle del webhook y de su contrato en [Webhook ws-inter](infraestructura/webhook-ws-inter.md).

---

## Request

### `/novedades-inter/solicitar`

Sin parámetros: la Step envía `body: "{}"`.

### `/webhooks/ws-inter/flujo-guia`

Un POST por guía con este sobre (contrato **confirmado** con callbacks reales del proveedor, 2026-10-19). También se aceptan arreglos de hasta 100 sobres:

```json
{
  "job_id": "790e7e46…",
  "guia": "240063616541",
  "ref": "6d638457…",
  "status": "ok",
  "result": {
    "guia": "240063616541",
    "consultado_en": "2026-10-19T12:04:28-05:00",
    "encontrado": true,
    "estado_actual": "Envío Admitido",
    "ultimo_motivo": "REHUSADO / SE NEGO A RECIBIR NOTIFICACION",
    "flujo": [ { "orden": 10, "motivo": "REHUSADO / …", "geo": { "latitud": 10.8690702, "longitud": -74.7854415, "url": "http://google.com/maps?q=…" } } ],
    "motivos": [ { "fecha": "2026-10-17T12:18:59-05:00", "estado": "En Proceso de Devolución", "motivo": "REHUSADO / …" } ],
    "remitente": { … }, "destinatario": { … }, "envio": { … }, "liquidacion": { … }
  },
  "error": null
}
```

- `ref` es la referencia que enviamos en la solicitud; si faltara, se correlaciona por el número de guía.
- **Si el proveedor no pudo consultar la guía** manda `status` distinto de `ok` (y/o `result: null`) con el motivo en `error`. Eso no es "guía no encontrada": la solicitud queda `Fallida` con el texto del error y se reintenta al vencer el refresco.
- De `result` solo se guardan `guia`, `consultado_en`, `encontrado`, `estado_actual`, `ultimo_motivo`, `flujo` y `motivos`. Los bloques personales y comerciales (`remitente`, `destinatario`, `envio`, `liquidacion`) **no se copian**: ya viven en `PedidosInter`.

---

## Response

### `/novedades-inter/solicitar` (200)

```json
{ "GuiasElegibles": 650, "SolicitudesEnviadas": 120, "SolicitudesFallidas": 0, "Saltadas": 530,
  "SeguimientosCerrados": 4, "MotivosReproyectados": 0, "PendientesPorTope": 0,
  "WsInterNoDisponible": false, "CredencialInvalida": false, "CorridaCortada": false, "Interrumpida": false }
```

### `/webhooks/ws-inter/flujo-guia`

| Código | Cuándo |
| ------ | ------ |
| `200`  | Recibido. Cuerpo: `{ Recibidos, Procesados, MotivosNuevos, Ignorados, Desconocidos, Invalidos, ConErrorProveedor }`. Un resultado inválido o de una guía que no pedimos **no** hace fallar la llamada |
| `400`  | El cuerpo no es JSON, está vacío, no es objeto/arreglo o trae más de 100 resultados. El proveedor **no** debe reintentar |
| `401`  | La API key no coincide (segunda barrera; la primera es API Gateway, que responde `403` sin clave) |
| `403`  | (API Gateway) falta o es inválida la API key |
| `500`  | Falla nuestra (p. ej. Mongo). El proveedor **debe reintentar**; el procesamiento es idempotente |

---

## Flujo interno

```
Step_Procesos_Logigho
  Lambda Pedidos Inter (legacy, sin cambios)
    -> Lambda Novedades Inter  ── /novedades-inter/solicitar ──> SolicitarNovedadesUseCase
         0. GET ws-inter /health            (si falla: alerta Slack y termina)
         1. PedidosInter: guías Inter en novedad
         2. cierra seguimientos que salieron de novedad
         3. re-escribe MotivoNovedadInter si la legacy lo pisó
         4. PlanificadorEnvios: ¿le toca consultar? -> POST ws-inter /webhook/flujo-guia {guia, ref}
         5. corta y alerta si fallan 5 seguidas o la API key es rechazada
    -> Lambda Proceso Guias Archivadas -> …

ws-inter ── POST callback (x-api-key) ──> API Gateway REST ──> RecibirResultadoUseCase
         valida -> busca el seguimiento por ref/guía -> ignora duplicados y viejos
         -> ExtractorMotivoNovedad -> $set PedidosInter.MotivoNovedadInter (solo si cambió)
         -> SeguimientoNovedadesInter (respuesta, contadores, historial)
```

Casos de uso: [Solicitar novedades](casos-uso/solicitar-novedades-usecase.md) ·
[Recibir resultado](casos-uso/recibir-resultado-usecase.md) · Modelos: [dominio/modelos.md](dominio/modelos.md).

---

## Dependencias externas

| Servicio | Uso |
| -------- | --- |
| `MongoDB` (Atlas en PreProd, DocumentDB en producción) | `PedidosInter` (lee estado, escribe `MotivoNovedadInter`) y `SeguimientoNovedadesInter` |
| `ws-inter` (`https://wsinter.duckdns.org`) | Proveedor externo: `GET /health` y `POST /webhook/flujo-guia` |
| `Slack` (webhook entrante) | Alertas inmediatas cuando ws-inter falla |
| `CloudWatch` | Logs JSON, métricas EMF (`Logigho/NovedadesInter`), alarmas y dashboard |

### Variables de entorno

Los valores sensibles van **cifrados con AES** (`EncripcionAES.EncryptAES256ECB`), como en las demás lambdas.

| Variable | Origen | Descripción |
| -------- | ------ | ----------- |
| `ENTORNO` | Terraform | `preprod` o `prod` (etiqueta de métricas y alertas) |
| `DATABASE_NAME` | Terraform | Base Mongo (`LogighoDB`) |
| `URL_WS_INTER` | Terraform | URL base del proveedor |
| `CADENA_CONEXION` | Pipeline (cifrada) | Conexión a Mongo |
| `API_KEY_WS_INTER` | Pipeline (cifrada) | Clave que enviamos a ws-inter en `x-api-key` |
| `WEBHOOK_API_KEY` | Pipeline (cifrada) | Clave que ws-inter debe enviar a nuestro webhook |
| `SLACK_WEBHOOK_ALERTAS` | Pipeline (cifrada) | Canal de alertas (opcional: sin él solo se escribe el log) |
| `ESTADOS_NOVEDAD` | Opcional | Estados de Inter que cuentan como novedad (por defecto: Intento de entrega, Telemercadeo, Reenvio, Devolucion Regional, Reclame en oficina) |
| `DESCRIPCIONES_NOVEDAD` | Opcional | Descripciones asociadas que cuentan (No logramos hacer la entrega, Tu envío presenta una novedad) |
| `HORAS_REFRESCO_ALTA` / `HORAS_REFRESCO_NORMAL` | Opcional | Cada cuántas horas se vuelve a consultar una guía sin cambios (6 / 24) |
| `MAX_GUIAS_POR_CORRIDA` | Opcional | Tope de envíos por corrida (1500). Bajarlo para la primera corrida en producción |
| `CONCURRENCIA_WS_INTER` / `FALLAS_SEGUIDAS_PARA_CORTAR` | Opcional | 4 / 5 |

---

## Historial de cambios

| Fecha | Autor | Cambio |
| ----- | ----- | ------ |
| 2026-10-18 | Iker Acevedo | Creación de la lambda, webhook, Terraform (`novedades`), pipelines y documentación |

---

## Observaciones

- **Contrato del callback confirmado.** El proveedor envía un sobre con el resultado dentro de `result` (ver Request). Todo lo que
  depende de esa forma está aislado en `Aplicacion/DTO/CallbackWsInterParser.cs`. Diagnóstico: el log `callback-con-descartados` incluye una muestra del cuerpo.
- **`estado_actual` del proveedor no es fiable** (en las muestras llega `Envío Admitido`, el primer estado del flujo, aunque la guía ya esté devuelta). No se usa; solo se usan `motivos` y `flujo`.
- **La detección de novedad de la lambda legacy no se reutiliza.** `ApiLambdaProcesoAutomaticoInter` usa los hijos de
  `PadreId == 28` del catálogo `EstadoGuiaLogighoNew` (solo "Custodia") y apenas 8 pedidos tienen `Estado Novedad`.
  La regla nueva vive en `ReglaSeleccionNovedad` y se configura por variables.
- **Carrera con la lambda legacy.** La legacy reescribe el pedido completo (`ReplaceOne`) cada 3 horas y podría borrar
  `MotivoNovedadInter` si coincide con un callback. Cada corrida de `solicitar` lo repone desde el seguimiento.
- **`Catch` oculta fallos de la Step.** Si la lambda no se puede invocar (p. ej. falta permiso en el rol de la Step) la
  ejecución termina `SUCCEEDED`. Lo detecta la alarma `corrida-ausente` (9 h sin ninguna corrida).
- **Índices.** Crear los tres índices (runbook, sección A2) antes de activar la Step (ver [runbook](infraestructura/runbook.md)).
- `$push` con `$slice` (tope de 50 en el historial) debe verificarse en DocumentDB en la primera corrida de producción.
