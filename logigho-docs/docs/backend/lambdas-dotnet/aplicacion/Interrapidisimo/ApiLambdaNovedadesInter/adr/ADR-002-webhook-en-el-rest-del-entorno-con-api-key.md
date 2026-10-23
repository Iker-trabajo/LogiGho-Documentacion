---

## Autor: Iker Acevedo  
Fecha creacion: 2026-10-18  
Estado: aceptada

# ADR-002 — Webhook en el REST único del entorno, protegido con API key

**Autor:** Iker Acevedo  
**Fecha:** 2026-10-18  
**Estado:** Aceptada

---

## Contexto

ws-inter necesita una URL fija a la cual devolver el resultado de cada guía. Debe ser alcanzable desde internet, estar
protegida y no depender de la sesión de ningún usuario (no hay Cognito).

La plataforma tiene **un solo REST por entorno** donde va todo endpoint nuevo:

| Entorno | REST | Stage | Quién lo administra |
| ------- | ---- | ----- | ------------------- |
| PreProd | `API_LogiGho_Rest_PreProd` (`8u5isw5u30`) | `PreProd` | Terraform, `environments/preprod/datos` (API, authorizer, deployment y stage) |
| Producción | `API_LogiGho_Rest` (`7elgyten88`) | `Produccion` | Nadie en Terraform (manual). Las rutas nuevas se crean por Terraform y el deployment se publica con el CLI |

## Opciones consideradas

### Opción A — Agregar la ruta al REST único del entorno

Pros: una sola superficie pública y un solo lugar donde buscar los endpoints; sigue la regla de la plataforma y el
precedente de `/webHookShopify` en producción. Contras: en PreProd el deployment pertenece al estado `datos`, así que ese
estado debe sumar los ids de la ruta a su trigger de redeploy y su `apply` (que exige la cadena Mongo de `datos`) es quien la publica.

### Opción B — API REST nuevo y propio del dominio (`Webhooks-LogiGho-<Entorno>`)

Pros: el estado del dominio sería dueño de todo. Contras: rompe la regla de un solo REST por entorno y multiplica las
superficies públicas. **Se probó y se descartó** (se creó en PreProd y se destruyó antes de usarse).

### Autenticación

**API key + usage plan** (elegida) frente a firma HMAC del cuerpo: el proveedor no documenta una firma; API Gateway valida
la clave **antes** de invocar la lambda, es un único secreto y se rota con `terraform apply -replace`. La lambda la vuelve a
comprobar en tiempo constante como segunda barrera. El método exige la clave y usa autorización `NONE`; los demás métodos
del REST siguen con su authorizer y no se ven afectados por el plan de uso. Se puede subir a HMAC + marca de tiempo si el
proveedor lo soporta.

## Decisión

**Se eligió:** Opción A con autenticación por API key.

**Razón:** un solo REST por entorno es la regla de la plataforma; la complejidad extra de Terraform (módulo de ruta
`novedades-webhook` que recibe `rest_api_id` y `parent_resource_id`, como el módulo de `datos`) es pequeña y conocida.

## Consecuencias

**Positivas:** una sola superficie pública; la ruta se encuentra junto a las demás; sin APIs que mantener aparte.  
**Negativas:**
- **PreProd:** crear la ruta exige un `apply` en `datos`; hay que usar el mismo valor de `TF_VAR_mongodb_connection_string`
  que ese dominio (si difiere, el plan cambia la lambda de reportes: revisar siempre el plan antes de aplicar).
- **Producción:** publicar el deployment con `create-deployment` publica **todo** lo pendiente del API, incluso cambios manuales
  ajenos sin publicar. Revisar antes de publicar y guardar el deployment anterior para revertir.
- Como el REST es compartido, las métricas de API Gateway por stage mezclan todas las rutas: la observabilidad del webhook
  se apoya en las métricas que publica la propia lambda (`CallbackErrorInterno`, `WebhookRechazado`, `CallbacksInvalidos`).
- La API key vive en el estado de Terraform (bucket S3 cifrado) y en el variable group. El REST de producción es *edge*:
  una ruta publicada tarda unos minutos en propagar (la prueba de humo reintenta ~2 min).

## Impacto en el código

| Módulo / Repo | Cambio |
| ------------- | ------ |
| `Terraform_Infraestructura_Logigho` | `modules/novedades-webhook` (ruta, permiso, API key, plan de uso); llamado desde `environments/preprod/datos` y `environments/prod/novedades` |
| `ApiLambdaNovedadesInter` | `ValidadorApiKey`, ruta `/webhooks/ws-inter/flujo-guia`, métrica `CallbackErrorInterno` |

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-10-18 | Iker Acevedo | Creación (opción B) |
| 2026-10-18 | Iker Acevedo | Reemplazada por la opción A: un solo REST por entorno |

## Referencias

- [Webhook y contrato con ws-inter](../infraestructura/webhook-ws-inter.md) · [API Gateway](../../../../../../arquitectura/api-gateway.md)
