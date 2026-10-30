# Métricas — Operación y despliegue

## Infraestructura (Terraform)

Toda la infra es código en `Terraform_Infraestructura_Logigho`:

- `modules/metricas-entrega-ciudad`: función Lambda (.NET 10, zip marcador + `ignore_changes` en código y variables, `prevent_destroy`), rol de mínimo privilegio (logs + autoinvocación + VPC opcional), log group, EventBridge Scheduler.
- `modules/metricas-rutas`: recursos REST `/metricas/entrega-ciudad[/corridas[/{id}]][/configuracion]`, métodos, integración `AWS_PROXY` (el `uri` es el `invoke_arn` de la lambda, sin doble envoltura), `OPTIONS` de CORS por recurso y el permiso de invocación.
- `environments/preprod/metricas` (REST `8u5isw5u30`, sin VPC, Atlas) y `environments/prod/metricas` (REST `7elgyten88`, padre `cyyv45wowa`, con VPC, DocumentDB).
- Rol OIDC `azdo-metricas-<entorno>` para el CD, con permiso solo sobre esta función.

| | PreProd | Producción |
|---|---|---|
| Cuenta | 376313750428 | 522814697657 |
| Base | MongoDB Atlas | DocumentDB (VPC privada) |
| Gateway | REST `8u5isw5u30`, stage `PreProd` | REST `7elgyten88`, stage `Produccion` |
| Red de la lambda | sin VPC | `vpc-0f1a9572609a839bc`, subred `subnet-0badcdb4420e95a4f`, grupo `sg-03c28184ccd5a836f` (la misma de inventario, datos y novedades de prod) |
| `terraform apply` | 44 recursos (2026-10-29) | **45 recursos** (2026-10-30; el extra es la política de acceso a la VPC) |

### Publicación del stage (importante)

En un **REST API v1**, crear recursos/métodos/integraciones **no los publica**: hay que crear un *deployment* nuevo y apuntarlo al stage, o el gateway responde **403 MissingAuthenticationToken** aunque la ruta exista.

- **PreProd:** el deployment lo gestiona Terraform en `environments/preprod/datos` (dueño del REST compartido), que suma `metricas_rutas.ids_para_despliegue` al trigger de su `aws_api_gateway_deployment` leyendo el state de métricas con `terraform_remote_state`. Hay que aplicar `environments/preprod/metricas` **antes** que `datos`.
- **Producción:** el stage `Produccion` de `7elgyten88` se publica **a mano con el CLI** (mismo patrón que inventario y novedades; el REST no está en Terraform, M-125/M-130/M-131):
  ```
  aws apigateway create-deployment --rest-api-id 7elgyten88 --stage-name Produccion --description "Metricas entrega por ciudad"
  ```
  Antes de publicar, revisar las rutas del REST (`get-resources`): el deployment publica **todo lo pendiente**, no solo métricas. Tras publicar puede tardar unos segundos en propagar (el humo dio 403 y luego 401).

## Scheduler

`cron(0 8 ? * SUN *)`, zona `America/Bogota` (domingo 8:00). Input `{"tipo":"CORRIDA","origen":"PROGRAMADA"}`. El alcance lo toma de `AlcanceProgramado` de la configuración.

!!! warning "El Scheduler de producción nace apagado"
    El módulo trae `estado_programacion = "ENABLED"` por defecto, pero `environments/prod/metricas/main.tf` lo fija en **`DISABLED`** (commit `db28f1a`). Motivo: con la configuración inicial ausente, la corrida del domingo habría calculado datos vacíos. Para activarlo, cambiar a `"ENABLED"` y volver a aplicar. Reversión: `"DISABLED"`.

## Pipelines (Azure DevOps)

Carpeta `Lambdas/Metricas/Entrega`:

| Pipeline | Dispara | Qué hace |
|---|---|---|
| `Lambdas-Metricas-EntregaCiudad-CI` | Build validation de `main`/`release/*` | compila, corre los 201 tests, empaqueta el zip. No toca AWS |
| `Lambdas-Metricas-EntregaCiudad-CD-PreProd` | push a `release/*` | sube código, añade variables aditivas, prueba de humo. **No** publica el stage (lo hace Terraform/datos) |
| `Lambdas-Metricas-EntregaCiudad-CD-Prod` | push a `main`, environment `produccion` (**aprobación manual**) | igual, contra prod |

Reutilizan `pipelines/scripts/lambda_release.py` sin modificarlo. Service connections OIDC `aws-preprod-metricas` / `aws-prod-metricas`; variable groups `metricas-preprod` / `metricas-prod-secretos`.

### `soloEnsayo`

La plantilla `plantillas/metricas-cd.yml` trae el parámetro `soloEnsayo`. Con `true` el pipeline empaqueta y hace un ensayo de **solo lectura** en AWS. Con `false` sube el zip a S3, respalda el código actual, añade las variables que faltan y actualiza la función. El primer despliegue de prod se hizo en dos pasadas: ensayo (run `#20261030.1`, verde) y luego `soloEnsayo: false` (commit `8642d4f`).

### Variables del variable group

| Variable | Valor | ¿Secreto? |
|---|---|---|
| `CADENA_CONEXION` | cadena Mongo cifrada AES (la misma de las demás lambdas del entorno) | **Sí (AES256)** |
| `DATABASE_NAME` | `LogighoDB` | No |
| `COLECCION_PEDIDOS` | `PedidosInter` | No |
| `COLECCION_USUARIOS` | `Users` | No |
| `NOMBRE_FUNCION` | `ApiLambdaMetricasEntregaCiudad` | No |
| `MAX_TIME_AGREGACION_SEG` | `600` | No |
| `COGNITO_REGION`, `COGNITO_USER_POOL_ID`, `COGNITO_CLIENT_IDS` | los del pool que emite los tokens del sitio (el mismo de las lambdas de inventario) | No (identificadores) |
| `STAGE_PREPROD` / `STAGE_PROD` | `PreProd` / `Produccion`: solo las usa el pipeline para armar la URL del humo | No |

El CD las añade de forma **aditiva**: nunca pisa un valor ya puesto en la función.

### Prueba de humo

`GET /metricas/entrega-ciudad` **sin token** debe responder **401** con `SIN_TOKEN`. Mientras el deployment del stage no ha propagado (o no se ha publicado) responde **403** y el pipeline marca el paso como fallido tras 12 intentos. En el **primer** despliegue de un dominio nuevo a prod esto es esperable: el código ya quedó desplegado y se resuelve publicando el stage (M-131).

## Datos que hay que preparar antes de la primera corrida

Los escribe Iker (la IA solo lee las bases):

1. **Índices:** `MetricasEntregaCiudadMes` por `{Mes:1}` (`mes`) y `{Mes:1,CorridaId:1}` (`mes_corrida`); `MetricasEntregaCorridas` por `{Inicio:-1}` (`inicio_desc`).
2. **Configuración inicial:** documento `_id="entrega-ciudad"` (versión 1, alcance `TODO`, mínimo 30, listas de estados, mapa de transportadoras, `XCARGO` excluida, `AjusteVisual` apagado). Sin él, la corrida clasificaría todo como "no cuenta".
3. **Módulos del menú:** `Modulo` 169 (Entregas) y 170 (Configuración Entregas); ver [Crear un módulo por base de datos](../../frontend/crear-modulo-db.md).

!!! warning "Nombre de la colección del resumen"
    La colección se llama **`MetricasEntregaCiudadMes`**. En producción se creó por error como `MetricasEntregaCiudad` (con los índices); se renombró antes de la primera corrida. Si el nombre no coincide, la corrida crea la correcta sola pero **sin índices**.

## Lanzar una corrida

**Desde el panel** (rol CEO/Desarrollador): pestaña *Corridas* → *Lanzar corrida*.

**Con AWS CLI** (sin token; usa el alcance de `AlcanceProgramado`). En PowerShell el JSON va en un archivo porque PowerShell le quita las comillas dobles:

```powershell
'{"tipo":"CORRIDA","origen":"PROGRAMADA"}' | Set-Content -Encoding ascii payload.json
aws lambda invoke --function-name ApiLambdaMetricasEntregaCiudad --region us-east-1 --invocation-type Event --cli-binary-format raw-in-base64-out --payload file://payload.json salida.json
```

`--invocation-type Event` es asíncrono (responde 202): una corrida `TODO` dura unos 2 minutos y el CLI síncrono se cortaría a los 60 s. Se registra como `PROGRAMADA` con `SolicitadoPor = scheduler`.

**Con Postman** (corrida `MANUAL`): `POST .../corridas` con el cuerpo `{"alcance":"TODO"}` y los encabezados `Token` (el `id_token` de la sesión) y `headersecurity`; luego `GET .../corridas/{id}` hasta ver `FINALIZADA`. La colección está en `specs/004_resumen_entrega_ciudad/pruebas/`.

La corrida escribe en `MetricasEntregaCorridas` (auditoría) y `MetricasEntregaCiudadMes` (resumen). Una corrida `EN_CURSO` de más de 20 minutos se considera caída y el candado se libera solo.

## Qué mirar tras una corrida

| Dato | Esperado |
|---|---|
| `Estado` | `FINALIZADA` |
| Duración | Menos de 10 min (la primera `TODO` en prod: 1 min 53 s) |
| `PorcentajeCiudadResuelta` | 95 % o más (primera en prod: 98,1 %) |
| `NoCuentaPorMotivo` | `SIN_CIUDAD` pequeño; `TRANSPORTADORA_DESCONOCIDA` casi cero |
| `DocumentosEscritos` | Un documento por `(mes, mun8, transportadora)` (25 023 en prod) |

## Reversión

- **Scheduler:** `estado_programacion = DISABLED` (Terraform) y ocultar la entrada del menú (poner `estado` distinto de `ACTIVO` en `Modulo` 169/170).
- **Ajuste visual:** apagarlo en el panel; no toca datos.
- **Código:** el CD respalda el código actual y deja los comandos de reversión en el log (`quitar-variables` para las variables nuevas).
- **Datos:** las colecciones `MetricasEntrega*` son independientes; borrarlas no afecta ningún otro flujo. `PedidosInter` nunca se escribe.
