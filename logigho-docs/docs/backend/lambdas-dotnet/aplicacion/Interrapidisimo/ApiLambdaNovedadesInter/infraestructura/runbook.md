## Autor: Iker Acevedo

Fecha creacion: 2026-10-18  
Estado: desarrollo

---

# Runbook: poner en marcha y operar novedades Inter

Qué hace cada pieza y en qué orden se enciende. **Nada de esto se ejecuta solo**: Terraform, los pipelines y los
scripts los lanza una persona. En producción cada paso es una escritura; léelos antes.

| Pieza | Dónde vive | Quién la crea |
| ----- | ---------- | ------------- |
| Lambda, rol, alarmas, dashboard, SNS | `Terraform_Infraestructura_Logigho` → `environments/<env>/novedades` + `modules/novedades-inter` | `terraform apply` (Iker) |
| Ruta del webhook en el REST del entorno + API key | `modules/novedades-webhook`, llamado desde `environments/preprod/datos` (PreProd) y `environments/prod/novedades` (producción) | `terraform apply` (Iker); en producción el deployment lo publica el CLI |
| Step y disparador de **PreProd** | `environments/preprod/procesos-automaticos/step_procesos_logigho.tf` | `terraform apply` (Iker) |
| Estado en la Step de **producción** | `pipelines/novedades/scripts/agregar_novedades_a_step_function.py` (la Step prod no está en Terraform) | Script (Iker) |
| Código y variables sensibles | `pipelines/novedades/` | Pipeline de Azure DevOps |
| Colección e índices | Tres `createIndex` (secciones A2 y B2) | `mongosh` (Iker) |

---

## A. PreProd

### A1. Infraestructura (Terraform) — en PowerShell

Son **tres directorios, en este orden** (el webhook vive en el REST de PreProd, que administra `datos`):

1. `environments/preprod/novedades`: la lambda, su rol, las alarmas y el dashboard.
2. `environments/preprod/datos`: agrega la ruta `/webhooks/ws-inter/flujo-guia` al REST `API_LogiGho_Rest_PreProd`, crea la API key y **redespliega el stage**.
3. `environments/preprod/procesos-automaticos`: la Step y su disparador (deshabilitado).

```powershell
cd C:\Users\Iker\Documents\LogiGho\Terraform_Infraestructura_Logigho
git switch feature/novedades-inter
$env:AWS_PROFILE = "Dev-logigho-test-01"        # cuenta 376313750428 (en PowerShell NO se usa "export")
aws sts get-caller-identity --query Account --output text     # debe imprimir 376313750428

terraform -chdir=environments/preprod/novedades init
terraform -chdir=environments/preprod/novedades plan "-out=novedades.tfplan"
terraform -chdir=environments/preprod/novedades apply novedades.tfplan
```

- `$env:AWS_PROFILE`: qué credenciales usa AWS en esa ventana. Si olvidas definirla, Terraform usa otra cuenta y su guardia `allowed_account_ids` lo frena.
- `-chdir=`: ejecuta Terraform dentro de esa carpeta sin moverte. `"-out=…"` va entre comillas porque PowerShell parte el argumento en el punto.
- `plan -out`: calcula qué cambiará y lo guarda. `apply <archivo>` ejecuta **exactamente** ese plan, no otro.
- Tras el `apply`, **confirma la suscripción del correo** (AWS envía un mail a cada dirección de `correos_alarma`).

> Si en este directorio ves `destroy` de recursos `aws_api_gateway_*` y de `aws_lambda_permission.api_gateway`, es el API
> temporal `Webhooks-LogiGho-PreProd` de una primera versión (descartado por ADR-002): se destruye y nadie lo usaba.

Luego `datos`. Este directorio también gestiona la lambda de reportes, así que **necesita el mismo valor** de la cadena Mongo
que ya usa (si no, el plan intentaría cambiarle las variables). Se lee de la propia lambda, sin imprimirla:

```powershell
$env:TF_VAR_mongodb_connection_string = aws lambda get-function-configuration --function-name ApiLambdaPublicarReporteAnalytics --query "Environment.Variables.MONGODB_CONNECTION_STRING" --output text

terraform -chdir=environments/preprod/datos init
terraform -chdir=environments/preprod/datos plan "-out=datos.tfplan"
```

**Revisa el plan antes de aplicar.** Debe mostrar: 9 recursos `novedades_webhook` por crear, `aws_api_gateway_deployment.preprod`
reemplazado (redeploy) y `aws_api_gateway_stage.preprod` actualizado. Resultado esperado: `Plan: 10 to add, 1 to change, 1 to destroy`.
**Si aparece `module.datos.aws_lambda_function.reportes` cambiando, detente**: el valor de la cadena no coincide.

```powershell
terraform -chdir=environments/preprod/datos apply datos.tfplan
Remove-Item Env:\TF_VAR_mongodb_connection_string          # no dejar el secreto en la sesión

terraform -chdir=environments/preprod/datos output novedades_url_webhook          # URL para el proveedor y para URL_WEBHOOK
terraform -chdir=environments/preprod/novedades output azure_devops_novedades_cd_rol_arn
```

La API key (secreta) se lee solo cuando se necesita: `terraform -chdir=environments/preprod/datos output -raw novedades_api_key_webhook`.

Por último la Step y su disparador (réplica de `Step_Procesos_Logigho`, **deshabilitado**):

```powershell
terraform -chdir=environments/preprod/procesos-automaticos init
terraform -chdir=environments/preprod/procesos-automaticos plan "-out=step.tfplan"
terraform -chdir=environments/preprod/procesos-automaticos apply step.tfplan
```

> El plan de este directorio también mostrará `aws_sfn_state_machine.procesos` "updated in-place". Es un cambio de
> **solo espacios en blanco** del JSON de la Step de TCC, anterior a este trabajo y sin efecto. Si prefieres no
> tocarlo: `apply -target=aws_iam_role.step_logigho -target=aws_iam_role_policy.step_logigho_invocar_lambdas -target=aws_sfn_state_machine.logigho -target=aws_iam_role.scheduler_logigho -target=aws_iam_role_policy.scheduler_logigho_iniciar_step -target=aws_scheduler_schedule.task_procesos_inter`.

### A2. Colección e índices

En `mongosh` (o la consola de Atlas / DBeaver), contra Atlas PreProd:

```javascript
use LogighoDB
db.SeguimientoNovedadesInter.createIndex({ Numeropreenvio: 1 }, { name: "ux_numeropreenvio", unique: true })
db.SeguimientoNovedadesInter.createIndex({ "Solicitud.Ref": 1 }, { name: "ix_solicitud_ref" })
db.SeguimientoNovedadesInter.createIndex({ EstadoSeguimiento: 1 }, { name: "ix_estado_seguimiento" })
db.SeguimientoNovedadesInter.getIndexes()
```

Crea la colección `SeguimientoNovedadesInter` y sus 3 índices. No afecta a `PedidosInter`.
Tanto Atlas PreProd como DocumentDB usan la base `LogighoDB`. Si la colección no existe, `createIndex` la crea.

### A3. Secretos cifrados y variable group

Los valores sensibles van cifrados con AES (igual que en las demás lambdas). Para cifrar uno:

```powershell
mkdir $env:TEMP\cifrar; cd $env:TEMP\cifrar
dotnet new console --force                      # proyecto de consola desechable, fuera del repo
dotnet add reference "C:\Users\Iker\Documents\LogiGho\Lambdas_NET_LogiGho\LambdasLogiGho.Infraestructura\Seguridad\Seguridad.csproj"
```

Reemplaza `Program.cs` por (lee de la entrada para que el secreto **no** quede en el historial de comandos):

```csharp
Console.Write(new Seguridad.Encripcion.EncripcionAES().EncryptAES256ECB(Console.ReadLine()));
```

Luego `dotnet run`, pega el valor en claro y Enter: imprime el texto cifrado.

Variable group **`novedades-inter-preprod-secretos`** (Azure DevOps → Pipelines → Library):

| Variable | Valor | ¿Secreta? |
| -------- | ----- | --------- |
| `CADENA_CONEXION` | Cadena de Atlas preprod, cifrada | Sí |
| `API_KEY_WS_INTER` | La clave que da el proveedor para llamar a ws-inter, cifrada | Sí |
| `WEBHOOK_API_KEY` | Salida `novedades_api_key_webhook` de `environments/preprod/datos`, cifrada | Sí |
| `URL_WEBHOOK` | Salida `novedades_url_webhook` de `environments/preprod/datos`, **sin cifrar** | No |
| `SLACK_WEBHOOK_ALERTAS` | (opcional en PreProd) webhook del canal de alertas, cifrado; si lo usas, agrégalo a `claves` en `cd-preprod.yml` | Sí |

### A4. Azure DevOps

1. Service connection AWS por OIDC llamada **`aws-preprod-novedades`** con el rol `azdo-novedades-preprod` (ARN en la salida de Terraform).
2. Pipelines desde `pipelines/novedades/`: `ci.yml` (nombre sugerido `Lambdas-Novedades-CI`, con política de rama como los demás),
   `cd-preprod.yml` (`Lambdas-Novedades-CD-PreProd`) y `cd-prod.yml` (`Lambdas-Novedades-CD-Prod`).
3. Una rama `release/…` con el código dispara `cd-preprod`: sube el zip, respalda, agrega las variables, publica el
   código, verifica el SHA y hace la prueba de humo (el webhook **sin** API key debe dar 403).

### A5. Prueba de punta a punta

```bash
# (bash; en PowerShell usa curl.exe y $URL = terraform … output -raw novedades_url_webhook)
URL=$(terraform -chdir=environments/preprod/datos output -raw novedades_url_webhook)
KEY=$(terraform -chdir=environments/preprod/datos output -raw novedades_api_key_webhook)

# 1) Sin clave: API Gateway rechaza (403)
curl -i -X POST "$URL" -d '{}'
# 2) Con clave y una guía que nunca pedimos: 200 con "desconocidos":1
curl -i -X POST "$URL" -H "x-api-key: $KEY" -H "Content-Type: application/json" -d '{"guia":"1","encontrado":false}'
# 3) Contrato real del proveedor, en modo síncrono (valida que la API key y el formato son los esperados)
curl -s -X POST https://wsinter.duckdns.org/webhook/flujo-guia/sync -H "x-api-key: <clave del proveedor>" \
     -H "Content-Type: application/json" -d '{"guia":"240063616541"}'
```

Luego, una corrida real: consola de **Step Functions → `Step_Procesos_Logigho` → Start execution** con entrada `{}`
(no hace falta activar el disparador). Revisa en Atlas:

```javascript
db.SeguimientoNovedadesInter.find({}, {Guia:1, Prioridad:1, "Solicitud.Estado":1, "UltimoMotivo.Motivo":1})
db.PedidosInter.find({ MotivoNovedadInter: { $exists: true } }, { Numeropreenvio:1, MotivoNovedadInter:1 })
```

Entrega al proveedor la URL y la API key (canal seguro), espera los callbacks y comprueba:
`MotivoNovedadInter` escrito, historial con 1 entrada y **una segunda ejecución con 0 envíos repetidos**
(`solicitudesEnviadas: 0` en el log `fin`). Para simular una caída pon temporalmente una `URL_WS_INTER` inválida
y verifica que llega la alerta de Slack.

Cuando todo esté bien, activa el disparador a mano: **EventBridge Scheduler → `TaskProcesosInter` → Enable**.
Terraform no lo vuelve a apagar.

---

## B. Producción

Todo es **nuevo y aditivo** (el plan de producción es 29 recursos a crear, 0 a cambiar, 0 a destruir; al REST existente solo se le agrega
una ruta), salvo el permiso y el estado en la Step. Orden:

1. **Webhook de Slack** del canal `#logigho-alertas-prod` (Slack → Apps → Incoming Webhooks). Cifrarlo (A3).
2. **Índices** en DocumentDB (túnel SSH al bastion, ver la memoria de conexiones):
   los mismos tres `createIndex` de A2 sobre la base `LogighoDB` de DocumentDB (versión 5.0):

   ```javascript
   use LogighoDB
   db.SeguimientoNovedadesInter.createIndex({ Numeropreenvio: 1 }, { name: "ux_numeropreenvio", unique: true })
   db.SeguimientoNovedadesInter.createIndex({ "Solicitud.Ref": 1 }, { name: "ix_solicitud_ref" })
   db.SeguimientoNovedadesInter.createIndex({ EstadoSeguimiento: 1 }, { name: "ix_estado_seguimiento" })
   db.SeguimientoNovedadesInter.getIndexes()
   ```
3. **Terraform** (`environments/prod/novedades`), con las credenciales de producción. En PowerShell:
   ```powershell
   $env:AWS_PROFILE = "<perfil prod>"            # cuenta 522814697657
   aws sts get-caller-identity --query Account --output text      # debe imprimir 522814697657
   terraform -chdir=environments/prod/novedades init
   terraform -chdir=environments/prod/novedades plan "-out=novedades.tfplan"
   ```
   El plan debe decir **`Plan: 29 to add, 0 to change, 0 to destroy`** (lambda, rol, alarmas, dashboard, SNS, rol OIDC y la
   ruta `/webhooks/ws-inter/flujo-guia` con su API key). **No debe tocar nada existente del REST `API_LogiGho_Rest`**: solo
   le agrega recursos nuevos. Si es correcto: `terraform -chdir=environments/prod/novedades apply novedades.tfplan` y
   confirma el correo de SNS. La Lambda queda en la VPC/subred/SG de `ApiLambdaProcesoAutomaticoInter` (ya tiene salida a
   internet por NAT y acceso a DocumentDB).
   > Terraform solo crea la ruta; **todavía no está publicada** en el stage. Se publica en el paso siguiente.

   **3b. Publicar el deployment del REST** (el stage `Produccion` no es de Terraform, igual que hace el pipeline de `datos`).
   Antes de publicar, anota el deployment actual (para revertir) y revisa que **no haya cambios manuales sin publicar**
   de otras personas en `API_LogiGho_Rest`, porque `create-deployment` publica TODO lo pendiente del API:
   ```powershell
   aws apigateway get-stage --rest-api-id 7elgyten88 --stage-name Produccion --query deploymentId --output text   # anotar (hoy: r5pecy)
   aws apigateway create-deployment --rest-api-id 7elgyten88 --stage-name Produccion --description "novedades ws-inter"
   ```
   Reversión: `aws apigateway update-stage --rest-api-id 7elgyten88 --stage-name Produccion --patch-operations op=replace,path=/deploymentId,value=<deployment anotado>`.
   Al ser un REST *edge* (CloudFront), la ruta tarda unos minutos en propagar: mientras tanto responde
   `403 {"message":"Missing Authentication Token"}`.
4. **Variable group** `novedades-inter-prod-secretos` (mismas variables que A3 más `SLACK_WEBHOOK_ALERTAS`) y service
   connection **`aws-prod-novedades`** (rol `azdo-novedades-prod`).
5. **Merge a `main`** → `cd-prod` (pide aprobación en el Environment `produccion`).
6. **Primera corrida acotada**: antes de tocar la Step, pon el tope en 50 guías:
   ```bash
   export AWS_PROFILE=<perfil prod>   # cuenta 522814697657
   MAX_GUIAS_POR_CORRIDA=50 python3 pipelines/scripts/lambda_release.py variables \
       --funcion ApiLambdaNovedadesInter --claves MAX_GUIAS_POR_CORRIDA
   ```
   (agrega la variable de forma **aditiva** y verifica que las demás quedaron intactas.)
7. **La Step** (ensayo primero, no escribe nada):
   ```bash
   export AWS_PROFILE=<perfil prod>
   python3 pipelines/novedades/scripts/agregar_novedades_a_step_function.py ensayo
   python3 pipelines/novedades/scripts/agregar_novedades_a_step_function.py aplicar
   ```
   `aplicar` guarda `respaldo-step-<fecha>.json`, agrega al rol de la Step una política que le permite invocar **solo**
   `ApiLambdaNovedadesInter` (sin ella el paso falla con `AccessDenied` y el `Catch` lo oculta) e inserta el estado después
   de `Lambda Pedidos Inter`. Para deshacerlo: `… revertir --archivo respaldo-step-<fecha>.json`.
8. **Vigila** la siguiente corrida de la Step (cada 3 h) o lanza una ejecución manual: logs, dashboard
   `Novedades-Inter-prod`, Slack. Verifica también que `$push` con `$slice` funciona en DocumentDB (el historial se llena).
9. **Entrega al proveedor** la URL de producción y su API key, y sube el tope:
   ```bash
   python3 pipelines/scripts/lambda_release.py quitar-variables --funcion ApiLambdaNovedadesInter --claves MAX_GUIAS_POR_CORRIDA
   ```
   (sin la variable vale el valor por defecto, 1500).

---

## C. Operación

### Cambiar un parámetro de ajuste o rotar un secreto

La fusión de variables del pipeline es **aditiva**: no cambia una variable que ya existe con otro valor. Para cambiar una:
`quitar-variables --claves NOMBRE` y luego `variables` (o redesplegar). Para rotar la API key del webhook:
`terraform apply -replace=module.novedades_webhook.aws_api_gateway_api_key.ws_inter` (en `preprod/datos` o `prod/novedades`; en producción hay que publicar de nuevo el deployment), cifrar la nueva, actualizar el variable
group, `quitar-variables` + redespliegue y entregarla al proveedor.

### Si llega una alerta

Ver la tabla de [observabilidad](observabilidad.md). La más importante, **ws-inter NO responde**: llamar al proveedor;
la corrida se reintenta sola cada 3 h y nada se pierde (las guías pendientes se envían cuando vuelva).

### Reversión

- **Código**: el pipeline imprime el comando de reversión (respaldo en `s3://logigho-artefactos-<env>/respaldo/ApiLambdaNovedadesInter/`).
- **Step de producción**: `agregar_novedades_a_step_function.py revertir --archivo …`.
- **Apagar solo esta función** sin tocar nada más: en la Step, quitar el estado con `revertir`, o dejar el disparador de PreProd deshabilitado.
- `MotivoNovedadInter` en `PedidosInter` es un campo extra: no rompe nada si se deja.
