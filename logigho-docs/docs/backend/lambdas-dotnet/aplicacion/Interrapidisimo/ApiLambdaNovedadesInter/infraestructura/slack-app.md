## Autor: Iker Acevedo

Fecha creacion: 2026-11-06  
Estado: desarrollo (PreProd hecha; producción pendiente)

---

# App de Slack «LogiGho» para los reportes de novedades

Paso a paso para crear la App que usa [`ApiLambdaNovedadesInterReportes`](../ApiLambdaNovedadesInterReportes.md): el bot sube los dos Excel al canal
y el comando `/reporte-novedades` los pide a mano. **Este documento no lleva tokens ni secretos**: los valores van cifrados con AES en el
variable group de Azure DevOps (ver [runbook](runbook.md#d-historia-027-distancias-y-reportes)).

> Marcas: los permisos y los campos salen del código (`ClienteSlack.cs`, `ReportesHandler.cs`) y de la spec 012; los nombres exactos de los menús de la
> consola de Slack pueden variar y **no se verificaron** aquí (hipótesis).

## Qué necesita la función de la App

| Necesidad | Origen en el código | Dónde se configura |
| --------- | ------------------- | ------------------ |
| Subir archivos al canal | `files.getUploadURLExternal` y `files.completeUploadExternal` | Scope de bot `files:write` |
| Publicar avisos de fallo | `chat.postMessage` | Scope de bot `chat:write` |
| Recibir el comando | `POST /novedades-inter/reportes/slack` | Scope `commands` y un *slash command* con su URL |
| Validar que la petición es de Slack | `X-Slack-Signature` y `X-Slack-Request-Timestamp` | *Signing secret* de la App |

Un webhook entrante **no** sirve: no sube archivos.

## Pasos

1. **Crear la App** «LogiGho» en el espacio de trabajo de Slack (desde cero).
2. **Scopes del bot** (*OAuth & Permissions*, *Bot Token Scopes*): `chat:write`, `files:write` y `commands`.
3. **Slash command**:
   - Nombre en producción: `/reporte-novedades`. En PreProd se probó con `/reporte-novedades-pruebas` (2026-11-06).
   - *Request URL*: la salida de Terraform, **por entorno**:
     - PreProd: `terraform -chdir=environments/preprod/datos output novedades_url_comando_slack`.
     - Producción: `terraform -chdir=environments/prod/novedades output url_comando_slack`.
   - Es `POST` de formulario (`application/x-www-form-urlencoded`) a `/novedades-inter/reportes/slack`. La ruta no tiene authorizer ni API key, porque Slack no puede enviarlas.
   - En prod la ruta existe en el REST solo cuando se publica el deployment con el CLI (runbook, sección B, paso 3b); antes responde `403 Missing Authentication Token`.
4. **Instalar la App** en el espacio de trabajo y copiar el **token del bot** (`xoxb-…`).
5. **Signing secret**: copiarlo de *Basic Information*.
6. **Cifrar ambos valores con AES** (runbook, A3) y guardarlos en el variable group `novedades-inter-<entorno>-secretos` como `SLACK_BOT_TOKEN` y `SLACK_SIGNING_SECRET`. Nunca en git, chat ni `docs/`.
7. **Canal**:
   - Crear (o elegir) el canal: uno de prueba para PreProd y el real para producción.
   - **Invitar al bot al canal** (si no, `files.completeUploadExternal` falla con `channel_not_found` o similar).
   - Guardar el **ID** del canal (`C…`, no su nombre) en `SLACK_CANAL_REPORTES` (sin cifrar).
8. **Usuarios permitidos**: guardar en `SLACK_USUARIOS_PERMITIDOS` los IDs de usuario de Slack (`U…` o `W…`) de quienes pueden usar el comando, separados por `;` o `,`. Lista vacía = nadie puede usarlo. Quien no esté en la lista recibe `403`.
9. **Correr el CD** de las 3 funciones para que las variables lleguen a `ApiLambdaNovedadesInterReportes`.

## Verificación

1. En el canal, ejecutar el comando con un usuario permitido: Slack debe mostrar al instante «Generando reportes…» (respuesta efímera) y en menos de 2 minutos deben llegar los 2 archivos
   `novedades-mas-500m-AAAA-MM-DD.xlsx` y `novedades-intentos-AAAA-MM-DD.xlsx` con un comentario resumen (CA10).
2. Con un usuario no permitido: respuesta de error (`403`); con una petición con firma alterada: `401` y la métrica `ComandoSlackFirmaRechazada`.
3. Estado observado: el comando de PreProd se probó de punta a punta en Slack (2026-11-06). **Producción: pendiente.**

## Seguridad

- La firma es `v0=HMAC_SHA256(signing_secret, "v0:{timestamp}:{cuerpo}")`; se compara en tiempo constante y la marca de tiempo debe estar a ±300 s.
- El cuerpo se limita a 4096 bytes y solo se lee como formulario **después** de validar la firma.
- Nunca se loguea el token, la firma ni el id del usuario.
- Para rotar el token o el signing secret: regenerarlo en Slack, cifrarlo, `quitar-variables` de esa clave y volver a correr el CD (las variables del CD son aditivas).

## Errores de Slack frecuentes

| Código de Slack | Causa probable |
| --------------- | -------------- |
| `channel_not_found` | El bot no está invitado al canal, o `SLACK_CANAL_REPORTES` es el nombre y no el ID |
| `invalid_auth` / `not_authed` | Token mal copiado o mal cifrado |
| `missing_scope` | Falta `files:write` o `chat:write` |

La función registra solo el código de Slack, nunca el token ni el cuerpo de la petición.
