# API Gateway

## Regla de la plataforma: un solo REST por entorno

Todo endpoint nuevo —incluidos los **webhooks de terceros**— se agrega al REST del entorno. **No se crean APIs REST separados**
por dominio ni por proveedor.

| Entorno | REST | Id | Stage | Tipo | Administración |
| ------- | ---- | -- | ----- | ---- | -------------- |
| PreProd | `API_LogiGho_Rest_PreProd` | `8u5isw5u30` | `PreProd` | Edge-optimized | Terraform: `environments/preprod/datos` es dueño del API, el authorizer Cognito, el deployment y el stage |
| Producción | `API_LogiGho_Rest` | `7elgyten88` | `Produccion` | Edge-optimized | **No está en Terraform** (manual). Las rutas nuevas se crean por Terraform (módulos que reciben `rest_api_id` y `parent_resource_id`) y el deployment se publica con el CLI |

> Producción tiene además `ApiIntegracionesExternas` (REST regional con `/pedidos` y `/token` para clientes), que no se usa
> para rutas nuevas de la plataforma.

### Cómo se agrega una ruta

1. Un módulo de Terraform con el árbol de recursos, el método, la integración Lambda proxy y el permiso de la Lambda
   (ejemplos: `modules/datos-reportes-analytics`, `modules/novedades-webhook`).
2. **PreProd:** el módulo se llama desde `environments/preprod/datos/main.tf` y sus `ids_para_despliegue` se suman al trigger de
   `aws_api_gateway_deployment.preprod`; el `apply` de ese directorio publica el stage.
3. **Producción:** el módulo se llama desde el entorno del dominio (`environments/prod/<dominio>`) con
   `rest_api_id = "7elgyten88"` y `parent_resource_id = "cyyv45wowa"`; después se publica con
   `aws apigateway create-deployment --rest-api-id 7elgyten88 --stage-name Produccion`. Antes de publicar, anotar el
   deployment actual (para revertir) y comprobar que no haya cambios manuales sin publicar, porque el deployment publica todo lo pendiente.

### Autenticación

| Tipo de ruta | Mecanismo |
| ------------ | --------- |
| Front y usuarios | Authorizer Cognito (`Autorizador`, header `Token`) |
| Webhooks de terceros (máquina a máquina) | API key en `x-api-key` con *usage plan* propio (límite de velocidad y cuota solo para esa clave); la lambda la vuelve a comprobar |

## Rutas registradas (las gestionadas por Terraform)

| Ruta | Servicio destino | Método | Auth |
|---|---|---|---|
| `/reportes-analytics/solicitar-carga` | `ApiLambdaPublicarReporteAnalytics` | POST | Cognito |
| `/reportes-analytics/confirmar-carga/{cargaId}` | `ApiLambdaPublicarReporteAnalytics` | POST | Cognito |
| `/webhooks/ws-inter/flujo-guia` | [`ApiLambdaNovedadesInter`](../backend/lambdas-dotnet/aplicacion/Interrapidisimo/ApiLambdaNovedadesInter/ApiLambdaNovedadesInter.md) (callback de ws-inter) | POST | API key |

Existen además rutas de producción creadas a mano (por ejemplo `/webHookShopify`) que aún no están documentadas aquí.
