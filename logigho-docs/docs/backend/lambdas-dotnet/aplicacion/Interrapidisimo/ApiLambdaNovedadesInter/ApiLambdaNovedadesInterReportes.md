## Autor: Iker Acevedo

Fecha creacion: 2026-11-06  
Estado: produccion (v8 desplegada 2026-11-08; Scheduler apagado)  
Actualizado: 2026-11-09 (v6 a v8)

---

## Lambda: ApiLambdaNovedadesInterReportes

**Trigger:** EventBridge Scheduler (programado, hoy apagado) + API Gateway REST (comando de Slack y enlaces de imágenes) + autoinvocación asíncrona  
**AOT:** No (runtime administrado `dotnet10`, 1024 MB, timeout 300 s)  
**Historia:** 027 · **Spec:** 012 · **Proyecto:** el mismo de [ApiLambdaNovedadesInter](ApiLambdaNovedadesInter.md) ([ADR-003](adr/ADR-003-distancias-y-reportes-en-el-mismo-proyecto.md))

> Marcas: **observado** (ruta citada), **Iker** (fecha), **hipótesis**. Esta página sigue el **código** (PR 635); las diferencias con la spec 012 están en [Observaciones](#observaciones).

---

## ¿Qué hace?

Arma dos Excel para la gerencia con ClosedXML y los sube a un canal de Slack con el bot:

1. **Reporte 1** (`novedades-mas-500m-AAAA-MM-DD.xlsx`): guías cuya última novedad se marcó a más de 500 m por calle de la dirección, **solo** las de Estado Logigho «Devolución completada/ratificada» o Estado Inter «En Proceso de Devolución» (v8). Cuatro hojas: Nuevas, Acumulado, Agencia u oficina y Por revisar (más de 50 km).
2. **Reporte 2** (`novedades-intentos-AAAA-MM-DD.xlsx`): guías en Estado Inter «En Proceso de Devolución» con un intento o menos cuyo motivo no es uno de los 8 «definitivos» y que **no** tienen registro en `AuditoriaCambioEntregaInter` (v8).

Ambos traen la dirección del pedido, la del mensajero, los días desde la primera novedad y las **imágenes** de la novedad («Ver imagen», hasta 10 por guía) con [enlace fijo firmado](infraestructura/evidencia-endpoint.md).

Las reglas están en [Reglas de los reportes](dominio/reglas-reportes.md). Esta página describe la función.

---

## Accionador

Una función, **cuatro formas de entrada** (observado: `ReportesHandler.cs`, `EventoReportes`):

| # | Origen | Entrada | Auth |
| - | ------ | ------- | ---- |
| 1 | EventBridge Scheduler `ApiLambdaNovedadesInterReportes-programado` | `{"origen":"programado"}` | IAM (rol del Scheduler, solo invoca esta función) |
| 2 | Slack, comando `/reporte-novedades` | `POST /novedades-inter/reportes/slack` (API Gateway, `application/x-www-form-urlencoded`) | **Sin authorizer ni API key**: firma HMAC de Slack + usuarios permitidos + WAF del REST |
| 3 | Autoinvocación de la ruta 2 | `{"origen":"slack","usuario":"<user_id>","nuevas":"completo"}` o `"nuevas":"desde","desde":"AAAA-MM-DD"` (opcionales) | IAM (permiso `lambda:InvokeFunction` solo sobre sí misma) |
| 4 | Enlaces de imágenes de los Excel (v8) | `GET /novedades/evidencia?k=…&f=…` (API Gateway) | **Sin authorizer ni API key**: firma HMAC-SHA256. Ver [la ruta](infraestructura/evidencia-endpoint.md) |

- Scheduler: `cron(0 8 ? * TUE,FRI *)` en zona `America/Bogota` (martes y viernes 8:00), reintentos 2 y edad máxima 3600 s
  (observado: `modules/novedades-inter/funciones-distancias-reportes.tf`).
- **Solo producción** (Iker 2026-11-06): en PreProd el Scheduler queda `DISABLED` a propósito (`reportes_programados_habilitados = false`);
  en prod también está **apagado hoy** (observado: `environments/prod/novedades/main.tf`, `reportes_programados_habilitados = false`) y se enciende con un PR de Terraform cuando Iker lo diga (mejora **M-271**).
- Handler: `ApiLambdaNovedadesInter::ApiLambdaNovedadesInter.Handlers.ReportesHandler::EjecutarAsync`.

---

## Request

### Comando de Slack

`POST /novedades-inter/reportes/slack`, cuerpo de formulario con (entre otros) `user_id` y `text` (el argumento). Cabeceras que Slack envía y la función exige:

| Cabecera | Uso |
| -------- | --- |
| `X-Slack-Request-Timestamp` | Segundos Unix; ventana de ±300 s |
| `X-Slack-Signature` | `v0=` + HMAC-SHA256 hexadecimal de `v0:{timestamp}:{cuerpo}` con el signing secret |

Orden de las defensas (deliberado, observado en `ReportesHandler.cs`):

1. Ruta y método correctos (si no: `404`).
2. Cuerpo presente y de **máximo 4096 bytes** (si no: `400`).
3. **Firma** (comparación en tiempo constante, `CryptographicOperations.FixedTimeEquals`). Inválida o vencida: `401`, y se emite la métrica `ComandoSlackFirmaRechazada`.
4. **Solo ahora** se lee el cuerpo como formulario y se toma `user_id`; debe cumplir `^[UW][A-Z0-9]{2,20}$` (si no: `400`).
5. `user_id` dentro de `SLACK_USUARIOS_PERMITIDOS` (si no: `403`, métrica `ComandoSlackUsuarioNoPermitido`).
6. **Argumento** (campo `text`, v6): vacío = «Nuevas» normal; `completo` o `desde-cero` = todas; `desde AAAA-MM-DD` = posteriores a esa fecha a las 00:00 de Bogotá (no futura). Lista blanca estricta; si es inválido responde la ayuda **sin generar nada** y sin repetir el texto recibido (200 efímero). Ver [Reglas de los reportes](dominio/reglas-reportes.md#nuevas-frente-a-acumulado-y-el-argumento-del-comando).
7. Se **autoinvoca de forma asíncrona** (`InvocationType=Event`) con la base ya normalizada y responde `200` de inmediato.

Nunca se loguea la firma, el secreto ni el id del usuario.

### Scheduler y autoinvocación

```json
{ "origen": "programado" }
{ "origen": "slack", "usuario": "U0000EJEMPLO" }
```

(El id es inventado.) Un `origen` distinto de `programado` o `slack` devuelve `400 origen-desconocido`.

---

## Response

### Comando de Slack (respuesta a Slack, en menos de 3 s)

```json
{ "response_type": "ephemeral", "text": "Generando reportes…" }
```

Si la autoinvocación falla, igualmente responde `200`, con el texto «No se pudo iniciar la generación de los reportes. Inténtalo de nuevo en un momento.»

| Código | Cuándo |
| ------ | ------ |
| `200` | Aceptado (o autoinvocación fallida, con texto efímero) |
| `400` | Evento vacío, cuerpo ausente, mayor de 4 KB, `user_id` ausente o inválido, origen desconocido |
| `401` | Firma inválida o vencida |
| `403` | Usuario no permitido |
| `404` | Ruta o método desconocido |

### Generación (Scheduler y autoinvocación)

Devuelve el resumen:

```json
{
  "CorridaId": "<id>",
  "Origen": "programado",
  "Estado": "Exitosa",
  "BaseNuevas": "normal",
  "Reporte1Nuevas": 3, "Reporte1Acumulado": 40, "Reporte1PorRevisar": 5, "Reporte1Agencia": 12,
  "Reporte2Nuevas": 7, "Reporte2Acumulado": 120, "Reporte2DescartadasOficina": 0,
  "CorridaNoRegistrada": false
}
```

(Valores inventados.) Métricas que emite: ver [Métricas](#metricas).

---

## Flujo interno

```
Scheduler / API Gateway ──> ReportesHandler
   comando: valida tamaño -> firma -> user_id -> usuario permitido -> IAutoinvocador (Event) -> 200 "Generando reportes…"
   generar: GenerarReportesUseCase
      1. base de «Nuevas»: normal (ObtenerUltimaExitosaAsync -> FechaInicio de esa corrida), completo o desde AAAA-MM-DD
      2. ObtenerFilasAsync(): SeguimientoNovedadesInter (de UltimaRespuesta solo estado_actual, motivos[].fecha y gestion_app[orden, s3_key])
                              + IntentosEntrega, Estado, Direccion y FechaCarga de PedidosInter (solo lectura)
      3. Reporte 1: FiltroEstadoReporte1 + (FiltroReporte1 | FiltroReporte1PorRevisar), separando agencia u oficina
         Reporte 2: FiltroReporte2 (Estado Inter, intentos, motivo) y descarte por AuditoriaCambioEntregaInter ($in por lotes de 1 000)
         «Nuevas» = UltimoMotivo.Fecha > base
      4. GeneradorExcelReportes: reporte 1 con 4 hojas; reporte 2 con 3 (Nuevas, Acumulado y Motivos); enlaces de imagen firmados
      5. ClienteSlack: files.getUploadURLExternal -> envío de los bytes a la upload_url -> files.completeUploadExternal
         (canal SLACK_CANAL_REPORTES, con un comentario resumen)
      6. RegistrarCorridaAsync -> ReportesNovedadesInter { Estado = Exitosa }
   si algo falla antes de subir: aviso corto al canal + corrida registrada como Fallida
```

- Si los archivos **ya llegaron** a Slack y falla el registro de la corrida, el estado sigue `Exitosa`, `CorridaNoRegistrada = true`,
  se avisa al canal y las «Nuevas» del próximo reporte pueden repetirse.
- El aviso de fallo usa su propio token de 10 s: si el fallo fue por agotar el tiempo, el token de la corrida ya está cancelado.
- Slack: el bot debe estar **invitado al canal** y `SLACK_CANAL_REPORTES` debe ser el **ID** del canal (`C…`), no su nombre.
  Pasos en [App de Slack](infraestructura/slack-app.md).

### Hojas y columnas

Encabezado en negrita, filtros, primera fila fija y anchos fijos. Fechas en la zona `ZONA_HORARIA` (por defecto `America/Bogota`). Las hojas de guías van de **menor a mayor «Días desde la primera novedad»**
(el día de la primera novedad es el 1). Detalle completo (columnas, orden, agencia u oficina, Motivos) en [Reglas de los reportes](dominio/reglas-reportes.md#orden-y-columnas-v8).

| Reporte | Hojas | Columnas |
| ------- | ----- | -------- |
| 1 | «Nuevas desde el último reporte», «Acumulado», «Agencia u oficina», «Por revisar (más de 50 km)» | 22 (+ «¿Nueva?» en la de agencia) + «Imagen novedad 1..N» |
| 2 | «Nuevas desde el último reporte», «Acumulado», «Motivos» | 15 + «Imagen novedad 1..N» |

Columnas comunes (14): Guía, Tienda, Seguimiento (Activo/Cerrado), Estado Inter, Estado Logigho, Último motivo, Fecha de la novedad, Fecha de creación del pedido, Días desde la creación del pedido,
**Días desde la primera novedad**, Intentos, Ciudad, Dirección del pedido y **Dirección del mensajero (texto)**. «Imagen novedad N» lleva el texto **«Ver imagen»** (máximo 10).

(Observado: `GeneradorExcelReportes.cs`.) **Cambio de la v6:** los Excel ya no son anónimos en la dirección: llevan la dirección del pedido y la del mensajero (canal privado de Slack) pero nunca nombre, teléfono, identificación ni nombre del mensajero.

### Sanitización del Excel

- Toda celda de texto que empieza por `=`, `+`, `-`, `@`, tabulación o retorno de carro lleva un `'` adelante (anti-inyección de fórmulas) y la celda se marca con formato de texto.
- Se eliminan caracteres de control y sustitutos sueltos, y se limita a 32 767 caracteres por celda.
- Los enlaces solo se escriben como hipervínculo si son `https://www.google.com/maps…` válidos en el host `www.google.com`; cualquier otro va como texto plano.

Observado: `SanitizadorExcel.cs`.

---

## Variables de entorno (solo nombres)

| Variable | Origen | ¿Cifrada? | Descripción |
| -------- | ------ | --------- | ----------- |
| `ENTORNO` | Terraform | No | `preprod` o `prod` |
| `DATABASE_NAME` | Terraform | No | Base Mongo |
| `CADENA_CONEXION` | Pipeline | Sí (AES) | Conexión a Mongo |
| `SLACK_BOT_TOKEN` | Pipeline | Sí (AES) | Token del bot de la App «LogiGho» |
| `SLACK_SIGNING_SECRET` | Pipeline | Sí (AES) | Valida la firma del comando |
| `SLACK_CANAL_REPORTES` | Pipeline | No | ID del canal (`C…`) |
| `SLACK_USUARIOS_PERMITIDOS` | Pipeline | No | IDs de usuario de Slack separados por `;` o `,`. Vacía = nadie puede usar el comando |
| `MOTIVOS_EXCLUIDOS_REPORTE2` | Opcional | No | Reemplaza la lista de 8 motivos (separada por `;`) |
| `ZONA_HORARIA` | Opcional | No | Por defecto `America/Bogota` |
| `SLACK_WEBHOOK_ALERTAS` | Pipeline | Sí (AES) | Alertas inmediatas (opcional) |
| `EVIDENCIAS_SECRETO_FIRMA` | Pipeline | Sí (AES256) | Clave HMAC de los enlaces de imágenes. Sin ella el reporte sale sin columnas de imagen. Rotarla invalida los enlaces ya enviados ([runbook D9](infraestructura/runbook.md#d9-rotar-evidencias_secreto_firma)) |
| `URL_EVIDENCIAS` | Pipeline | No | URL https base de `GET /novedades/evidencia`, sin query (salida `url_evidencias` de Terraform) |
| `EVIDENCIAS_BUCKET` | Terraform y Pipeline | No | Bucket de las fotos (`wsinter-evidencias-prod` en prod) |
| `EVIDENCIAS_REGION` | Opcional | No | Región del bucket para la presignación (por defecto `AWS_REGION`) |

---

## Dependencias externas

| Servicio | Uso |
| -------- | --- |
| `MongoDB` | Lee `SeguimientoNovedadesInter`, `PedidosInter` y `AuditoriaCambioEntregaInter` (solo lectura); escribe `ReportesNovedadesInter` |
| `Amazon S3` | `GetObject` presignado de `gestion-app/*` en el bucket de evidencias (permiso `s3:GetObject` solo sobre ese prefijo, Terraform). Ver [la ruta de evidencias](infraestructura/evidencia-endpoint.md) |
| `Slack API` (`https://slack.com/api/`) | `files.getUploadURLExternal`, `files.completeUploadExternal`, `chat.postMessage` con el bot token; un webhook no sube archivos |
| `AWS Lambda` | Autoinvocación asíncrona (permiso solo sobre sí misma) |
| `API Gateway REST` | Ruta `POST /novedades-inter/reportes/slack` (módulo `novedades-reportes-slack`): PreProd REST `8u5isw5u30`; prod REST `7elgyten88` (el deployment lo publica Iker con el CLI) |

---

## Métricas

Namespace `Logigho/NovedadesInter`, `Operacion = reportes` (observado: `ReportesHandler.cs`): `ReportesExitosos`, `ReportesFallidos`,
`ReportesNoRegistrados`, `FilasReporte1Nuevas`, `FilasReporte1Acumulado`, `FilasReporte2Nuevas`, `FilasReporte2Acumulado`, `FilasReporte2DescartadasOficina` (v8),
`ComandoSlackFirmaRechazada`, `ComandoSlackUsuarioNoPermitido`.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-11-06 | Iker Acevedo | Creación (PR de Lambdas 619, 622, 623, 624). Comando probado de punta a punta en Slack de PreProd |
| 2026-11-09 | Iker Acevedo (redactado con IA) | v6 a v8: Estado Inter y argumento del comando, agencia u oficina, filtro de estado, auditoría de oficina, dirección del mensajero, imágenes firmadas, ruta de evidencias (PR 635, 636; desplegada 2026-11-08) |

---

## Observaciones

- **Medido en PreProd (2026-11-06):** la generación tardó ~3 s y usó ~157 MB de 1024. **Pendiente medir en frío** que el comando
  responda en menos de 3 s (CA10, mejora **M-259**); bajar memoria a 512 (**M-261**).
- **Memoria con volumen:** lee los seguimientos casi completos en cada corrida; vigilar memoria y duración cuando la colección pase de ~50 000 (**M-257**).
- **Discrepancia con la spec:** la spec 012 §2.2 dice que los bytes se envían con `PUT`; el código usa `POST` hacia la `upload_url`
  (`ClienteSlack.cs`). La documentación sigue al código.
- **Frontera de «Nuevas»:** el código usa el **inicio** (`FechaInicio`) de la última corrida exitosa; la spec dice «fecha de la última corrida exitosa». Con `completo` o `desde AAAA-MM-DD` no se consulta la última corrida.
- **Verificado en prod 2026-11-08 (observado, ficha del dominio):** los Excel trajeron dirección del mensajero (100 %), imágenes con enlace firmado y 0 guías del reporte 2 en la auditoría de oficina.
- **Limpieza:** el script `tools/reportes-novedades/limpiar_ejecuciones_reportes.py` (lo ejecuta Iker) borra `ReportesNovedadesInter` para que el siguiente reporte arranque de cero.
- **Spec 012 desactualizada en dos puntos:** §5.4 (orden del reporte 1 y número de columnas) y §1/CA1 (ruta solo a pie). Manda el código; ver [Reglas de los reportes](dominio/reglas-reportes.md#discrepancias-con-la-spec-012).
- **Alertas (M-265):** el CD de prod no pasa `SLACK_WEBHOOK_ALERTAS` a Reportes; sus avisos van por el bot al canal.
- **Nombre del comando por entorno:** en PreProd se probó con `/reporte-novedades-pruebas` (observado en `tasks.md` de la spec, 2026-11-06); en prod será `/reporte-novedades`.
