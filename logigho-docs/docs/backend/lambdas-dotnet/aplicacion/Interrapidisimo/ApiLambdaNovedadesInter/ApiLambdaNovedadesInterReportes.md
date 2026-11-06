## Autor: Iker Acevedo

Fecha creacion: 2026-11-06  
Estado: desarrollo (PreProd validada 2026-11-06; producción pendiente)

---

## Lambda: ApiLambdaNovedadesInterReportes

**Trigger:** EventBridge Scheduler (programado) + API Gateway REST (comando de Slack) + autoinvocación asíncrona  
**AOT:** No (runtime administrado `dotnet10`, 1024 MB, timeout 300 s)  
**Historia:** 027 · **Spec:** 012 · **Proyecto:** el mismo de [ApiLambdaNovedadesInter](ApiLambdaNovedadesInter.md) ([ADR-003](adr/ADR-003-distancias-y-reportes-en-el-mismo-proyecto.md))

> Marcas: **observado** (ruta citada), **Iker** (fecha), **hipótesis**. Producción no tiene aún esta función: va como **pendiente**.

---

## ¿Qué hace?

Arma dos Excel para la gerencia con ClosedXML y los sube a un canal de Slack con el bot:

1. **Reporte 1** (`novedades-mas-500m-AAAA-MM-DD.xlsx`): guías cuya última novedad se marcó a más de 500 m por calle de la dirección.
2. **Reporte 2** (`novedades-intentos-AAAA-MM-DD.xlsx`): guías con un intento o menos cuyo motivo no es uno de los 8 «definitivos».

Las reglas están en [Reglas de los reportes](dominio/reglas-reportes.md). Esta página describe la función.

---

## Accionador

Una función, **tres formas de entrada** (observado: `ReportesHandler.cs`, `EventoReportes`):

| # | Origen | Entrada | Auth |
| - | ------ | ------- | ---- |
| 1 | EventBridge Scheduler `ApiLambdaNovedadesInterReportes-programado` | `{"origen":"programado"}` | IAM (rol del Scheduler, solo invoca esta función) |
| 2 | Slack, comando `/reporte-novedades` | `POST /novedades-inter/reportes/slack` (API Gateway, `application/x-www-form-urlencoded`) | **Sin authorizer ni API key**: firma HMAC de Slack + usuarios permitidos + WAF del REST |
| 3 | Autoinvocación de la ruta 2 | `{"origen":"slack","usuario":"<user_id>"}` | IAM (permiso `lambda:InvokeFunction` solo sobre sí misma) |

- Scheduler: `cron(0 8 ? * TUE,FRI *)` en zona `America/Bogota` (martes y viernes 8:00), reintentos 2 y edad máxima 3600 s
  (observado: `modules/novedades-inter/funciones-distancias-reportes.tf`).
- **Solo producción** (Iker 2026-11-06): en PreProd el Scheduler queda `DISABLED` a propósito (`reportes_programados_habilitados = false`);
  en prod también **nace apagado** y se enciende en un segundo apply, en la salida a producción. **Pendiente.**
- Handler: `ApiLambdaNovedadesInter::ApiLambdaNovedadesInter.Handlers.ReportesHandler::EjecutarAsync`.

---

## Request

### Comando de Slack

`POST /novedades-inter/reportes/slack`, cuerpo de formulario con (entre otros) `user_id`. Cabeceras que Slack envía y la función exige:

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
6. Se **autoinvoca de forma asíncrona** (`InvocationType=Event`) y responde `200` de inmediato.

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
  "Reporte1Nuevas": 3, "Reporte1Acumulado": 40,
  "Reporte2Nuevas": 7, "Reporte2Acumulado": 120,
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
      1. IRepositorioReportes.ObtenerUltimaExitosaAsync()  -> frontera de «Nuevas» (FechaInicio de esa corrida)
      2. ObtenerFilasAsync(): SeguimientoNovedadesInter (sin UltimaRespuesta, Historial, Solicitud ni Distancias)
                              + IntentosEntrega de PedidosInter (solo lectura)
      3. FiltroReporte1 / FiltroReporte2  -> filas de cada reporte; «Nuevas» = UltimoMotivo.Fecha > frontera
      4. GeneradorExcelReportes: 2 libros, cada uno con 2 hojas
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

Cada Excel tiene la hoja «Nuevas desde el último reporte» y la hoja «Acumulado», con encabezado en negrita, filtros, primera fila
fija y anchos fijos. Orden de las filas: fecha de novedad descendente. Fechas en la zona `ZONA_HORARIA` (por defecto `America/Bogota`).

**Reporte 1** — Guía · Tienda · Estado seguimiento · Estado Inter · Último motivo · Fecha novedad (Bogotá) · Intentos · Ciudad ·
Metros por calle · Metros línea recta · Precisión dirección · Ruta en Maps · Ubicación del mensajero.

**Reporte 2** — Guía · Tienda · Estado seguimiento · Estado Inter · Último motivo · Fecha novedad (Bogotá) · Intentos · Ciudad ·
Metros por calle · Color · Ruta en Maps.

(Observado: `GeneradorExcelReportes.cs`.) Sin nombre, teléfono ni dirección del destinatario.

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

---

## Dependencias externas

| Servicio | Uso |
| -------- | --- |
| `MongoDB` | Lee `SeguimientoNovedadesInter` y `PedidosInter` (solo lectura); escribe `ReportesNovedadesInter` |
| `Slack API` (`https://slack.com/api/`) | `files.getUploadURLExternal`, `files.completeUploadExternal`, `chat.postMessage` con el bot token; un webhook no sube archivos |
| `AWS Lambda` | Autoinvocación asíncrona (permiso solo sobre sí misma) |
| `API Gateway REST` | Ruta `POST /novedades-inter/reportes/slack` (módulo `novedades-reportes-slack`): PreProd REST `8u5isw5u30`; prod REST `7elgyten88` (el deployment lo publica Iker con el CLI) |

---

## Métricas

Namespace `Logigho/NovedadesInter`, `Operacion = reportes` (observado: `ReportesHandler.cs`): `ReportesExitosos`, `ReportesFallidos`,
`ReportesNoRegistrados`, `FilasReporte1Nuevas`, `FilasReporte1Acumulado`, `FilasReporte2Nuevas`, `FilasReporte2Acumulado`,
`ComandoSlackFirmaRechazada`, `ComandoSlackUsuarioNoPermitido`.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-11-06 | Iker Acevedo | Creación (PR de Lambdas 619, 622, 623, 624). Comando probado de punta a punta en Slack de PreProd |

---

## Observaciones

- **Medido en PreProd (2026-11-06):** la generación tardó ~3 s y usó ~157 MB de 1024. **Pendiente medir en frío** que el comando
  responda en menos de 3 s (CA10, mejora **M-259**); bajar memoria a 512 (**M-261**).
- **Memoria con volumen:** lee los seguimientos casi completos en cada corrida; vigilar memoria y duración cuando la colección pase de ~50 000 (**M-257**).
- **Discrepancia con la spec:** la spec 012 §2.2 dice que los bytes se envían con `PUT`; el código usa `POST` hacia la `upload_url`
  (`ClienteSlack.cs`). La documentación sigue al código.
- **Frontera de «Nuevas»:** el código usa el **inicio** (`FechaInicio`) de la última corrida exitosa; la spec dice «fecha de la última corrida exitosa».
- **Nombre del comando por entorno:** en PreProd se probó con `/reporte-novedades-pruebas` (observado en `tasks.md` de la spec, 2026-11-06); en prod será `/reporte-novedades`.
