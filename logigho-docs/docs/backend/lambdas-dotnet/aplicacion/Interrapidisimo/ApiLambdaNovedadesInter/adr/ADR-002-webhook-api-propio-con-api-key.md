---

## Autor: Iker Acevedo  
Fecha creacion: 2026-10-18  
Estado: aceptada

# ADR-002 — Webhook en un API REST propio del dominio, protegido con API key

**Autor:** Iker Acevedo  
**Fecha:** 2026-10-18  
**Estado:** Aceptada

---

## Contexto

ws-inter necesita una URL fija a la cual devolver el resultado de cada guía. Debe ser alcanzable desde internet, estar
protegida y no depender de la sesión de ningún usuario (no hay Cognito).

En AWS ya existen `API_LogiGho_Rest` (producción, manual, no está en Terraform), `ApiIntegracionesExternas` (producción, para
clientes, manual) y `API_LogiGho_Rest_PreProd` (PreProd, el estado `datos` es dueño de su deployment).

## Opciones consideradas

### Opción A — Agregar la ruta a un API existente

Pros: ninguna URL nueva. Contras: en producción habría que editar a mano un API que no está en Terraform; en PreProd el
deployment pertenece al estado `datos`, por lo que `novedades` dependería de otro estado para publicar su ruta.

### Opción B — API REST nuevo y propio del dominio (`Webhooks-LogiGho-<Entorno>`)

Pros: Terraform es dueño de todo (rutas, deployment, stage, API key, usage plan); cero riesgo sobre APIs en uso; las
ráfagas del proveedor no comparten límites con el front; futuros webhooks de terceros se agregan como rutas
`/webhooks/<proveedor>/…`. Contras: un API más.

### Autenticación

**API key + usage plan** (elegida) frente a firma HMAC del cuerpo: el proveedor no documenta una firma; la API key la
valida API Gateway **antes** de invocar la lambda, es un único secreto y se rota con `terraform apply -replace`. La lambda
la vuelve a comprobar en tiempo constante como segunda barrera. Se puede subir a HMAC + marca de tiempo si el proveedor lo soporta.

## Decisión

**Se eligió:** Opción B con autenticación por API key.

**Razón:** aislamiento y propiedad total del estado; la API key corta el tráfico no autorizado sin costo de invocar la lambda.

## Consecuencias

**Positivas:** sin cambios sobre infraestructura existente; throttling y cuota propios; métricas por ruta.  
**Negativas:** la API key vive en el estado de Terraform (bucket S3 cifrado) y en el variable group; los 4xx por clave
inválida se miden con una alarma, no se bloquean por IP.

## Impacto en el código

| Módulo / Repo | Cambio |
| ------------- | ------ |
| `Terraform_Infraestructura_Logigho` | `modules/novedades-inter/webhook.tf` |
| `ApiLambdaNovedadesInter` | `ValidadorApiKey`, ruta `/webhooks/ws-inter/flujo-guia` |

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-10-18 | Iker Acevedo | Creación |

## Referencias

- [Webhook y contrato con ws-inter](../infraestructura/webhook-ws-inter.md)
