---
autor: Iker Acevedo
fecha_creacion: 2026-10-26
ultima_actualizacion: 2026-10-26
estado: produccion
---

# CI/CD y despliegue — Servicio Público / ConsultarPedidos

Montaje de integración y despliegue continuo del sub-dominio `ServicioPublico/ConsultarPedidos`, que cubre dos lambdas .NET: `APILambdaConsultarPedidos` y `APILambdaObtenerToken`.

---

## Pipelines

Viven en el repo de lambdas, bajo `pipelines/servicio-publico/consultar-pedidos/`. El sub-dominio es **autocontenido**: sus plantillas (CI y CD) están en su propia carpeta `plantillas/`, no dependen de las de `pedidos`.

| Pipeline | Archivo | Dispara con | Despliega a |
| --- | --- | --- | --- |
| CI | `ci.yml` → `plantillas/servicio-publico-ci.yml` | PR hacia `main` (build validation) | — (solo compila/prueba/empaqueta) |
| CD PreProd | `cd-preprod.yml` → `plantillas/servicio-publico-cd.yml` | push a `release/*` | AWS PreProd (`376313750428`) |
| CD Prod | `cd-prod.yml` → `plantillas/servicio-publico-cd.yml` | merge a `main` | AWS Prod (`522814697657`), con aprobación del Environment `produccion` |

### Qué hace el CI

- Compila las 2 lambdas en Release.
- Corre los 11 tests (fechas + filtrado de tiendas).
- Empaqueta el zip de cada lambda.

### Qué hace el CD (paso a paso)

1. Empaqueta el zip de cada lambda.
2. Lo sube a S3: `releases/<función>/<sha>.zip`.
3. Respalda el código actual (para revertir).
4. Añade las variables de entorno que falten (modo **aditivo**, nunca pisa las existentes).
5. Publica el código nuevo (`UpdateFunctionCode`).
6. Espera a que AWS confirme `Active/Successful`.
7. Prueba de humo: `401` sin token.

El script compartido que ejecuta el release es `pipelines/scripts/lambda_release.py`.

---

## Flujo de ramas → entornos

```
feature/*  --PR-->  release/*          --push-->  CD PreProd  -->  AWS PreProd
                        |
                        +--PR-->  main  --merge-->  CD Prod  --(aprobación)-->  AWS Prod
```

- `release/*` despliega **solo** a PreProd.
- `main` despliega a **Producción**, y el Environment `produccion` frena el deploy hasta la aprobación manual.
- El pipeline lee el YAML **de la rama que ejecuta**: una corrección del pipeline debe llegar a esa misma rama.

---

## Roles y service connections

Los roles los crea Terraform; el service connection de DevOps debe apuntar al ARN del rol correcto (lo asume vía OIDC).

| Entorno | Service connection (DevOps) | Rol AWS (Role to Assume) |
| --- | --- | --- |
| PreProd | `aws-preprod-servicio-publico-consultar-pedidos` | `arn:aws:iam::376313750428:role/azdo-servicio-publico-preprod` |
| Prod | `aws-prod-servicio-publico-consultar-pedidos` | `arn:aws:iam::522814697657:role/azdo-servicio-publico-prod` |

El permiso del rol está acotado a `UpdateFunctionCode` + variables de entorno de las dos lambdas, y a sus carpetas S3 de artefactos (`logigho-artefactos-<entorno>/releases/<función>/*`).

!!! note "Secretos"
    Los valores de variables/secretos de las lambdas viven en **variable groups de Azure Library**. En el código y en los pipelines van solo los **nombres**, nunca los valores. Los `aws-lambda-tools-defaults.json` quedan con `environment-variables` vacío.

---

## Bitácora del estreno (3 fallos reales)

La primera corrida de un pipeline nuevo suele tropezar con cosas de conexión/configuración que no aparecen en la máquina local. Estos fueron los tres y su arreglo:

### 1. CI: plantilla no encontrada

```
File /pipelines/templates/lambda-dotnet-ci-varias.yml not found in repository
```

`ci.yml` apuntaba a `../../templates/lambda-dotnet-ci-varias.yml`. Esa plantilla (a) está a 3 niveles, no 2, y (b) ni siquiera es compartida: vive en `pipelines/pedidos/plantillas/`, es local de pedidos.

**Arreglo:** se creó la plantilla de CI propia del sub-dominio (`plantillas/servicio-publico-ci.yml`) y el `ci.yml` apunta a ella. El sub-dominio queda autocontenido.

### 2. CD: `length()` no soportado

```
servicio-publico-cd.yml (Line: 127): Kind 'Object' not supported.
Only arrays, strings, dictionaries... are supported for the length function.
```

Azure rechaza `gt(length(lambda.claves), 0)` dentro de un bucle `each lambda`: ahí ve `lambda.claves` como `Object`, no como lista. La plantilla original usaba `length(parameters.claves)` a nivel raíz (donde sí es lista).

**Arreglo:** como ambas lambdas **siempre** tienen claves (ConsultarPedidos 4, ObtenerToken 1), el guard era innecesario. Se quitó en las dos ocurrencias (ensayo y variables) y se dejaron los pasos directos.

### 3. CD: rol equivocado (AccessDenied en S3)

```
role assumed via OIDC Token: arn:aws:sts::376313750428:assumed-role/azdo-guias-preprod/...
upload failed ... s3://logigho-artefactos-preprod/releases/APILambdaConsultarPedidos/...
AccessDenied ... not authorized to perform: s3:PutObject
```

El CD apuntaba al service connection viejo `aws-preprod-servicio-publico`, que está enganchado al rol de **guías** (`azdo-guias-preprod`). Ese rol no tiene permiso sobre las carpetas S3 de consultar-pedidos.

**Arreglo:** los CD apuntan al service connection propio `aws-{preprod,prod}-servicio-publico-consultar-pedidos`, que asume el rol `azdo-servicio-publico-{preprod,prod}` con el permiso correcto.

---

## Verificación post-deploy (PreProd)

Confirmado contra AWS tras el primer deploy verde:

- Ambas lambdas actualizadas, estado `Active`.
- `APILambdaConsultarPedidos` con sus 4 variables, incluida `FECHA_MINIMA_PEDIDOS` (añadida por el modo aditivo; antes no estaba).
- `APILambdaObtenerToken` con su `COGNITO_CLIENT_ID`.

La prueba funcional de la API está en [Prueba PreProd](../Guias/prueba-preprod.md) (rutas, authorizer, curls).

---

## Diferencias PreProd / Prod

| | PreProd | Prod |
| --- | --- | --- |
| Cuenta AWS | `376313750428` | `522814697657` |
| Trigger | push a `release/*` | merge a `main` |
| Aprobación | ninguna (deploy directo) | Environment `produccion` (manual) |
| `function-name` ConsultarPedidos | `APILambdaConsultarPedidos` | (verificar nombre real en Prod) |

Solo cambian cuenta, nombres y aprobación: el YAML y la plantilla son los mismos. Si PreProd despliega limpio, Prod hace el mismo camino con la aprobación de por medio.
