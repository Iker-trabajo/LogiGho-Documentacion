## Autor: Iker Acevedo

Fecha creacion: 2026-10-18  
Estado: desarrollo

---

# Webhook y contrato con ws-inter

## El proveedor (hechos verificados en `https://wsinter.duckdns.org/openapi.json`, v2.0.0)

| Endpoint | Descripción |
| -------- | ----------- |
| `GET /health` | ¿Está vivo? Lo usamos antes de cada corrida |
| `POST /webhook/flujo-guia` | **Una guía por llamada**: `{ "guia": "…", "ref": "…" }`, header `x-api-key`. Responde `202 { job_id, guia, status:"queued" }`. El resultado llega después por callback |
| `POST /webhook/flujo-guia/sync` | Igual, pero devuelve la respuesta directo. Útil para probar y depurar |

El proveedor usa **una URL de callback fija** porque con muchas guías no alcanzan 30 s por solicitud.

## Nuestro webhook

`POST /webhooks/ws-inter/flujo-guia` en el **REST único del entorno** (regla de la plataforma: ver
[API Gateway](../../../../../../arquitectura/api-gateway.md) y [ADR-002](../adr/ADR-002-webhook-en-el-rest-del-entorno-con-api-key.md)):

| Entorno | API | Stage | URL base |
| ------- | --- | ----- | -------- |
| PreProd | `API_LogiGho_Rest_PreProd` (`8u5isw5u30`) | `PreProd` | `https://8u5isw5u30.execute-api.us-east-1.amazonaws.com/PreProd` |
| Producción | `API_LogiGho_Rest` (`7elgyten88`) | `Produccion` | `https://7elgyten88.execute-api.us-east-1.amazonaws.com/Produccion` |

La URL completa es la base + `/webhooks/ws-inter/flujo-guia`. También sale de Terraform:
`novedades_url_webhook` en `environments/preprod/datos` y `url_webhook` en `environments/prod/novedades`.

**Seguridad.** El método exige API key (`x-api-key`): sin ella API Gateway responde `403 {"message":"Forbidden"}` sin invocar
la lambda. La lambda la vuelve a comprobar (comparación en tiempo constante). El *usage plan* limita a 20 peticiones/s
(ráfaga 40) y 50 000 al día **solo para quien use esa clave**; los demás métodos del REST siguen con su authorizer y no se
ven afectados. La clave la genera Terraform (`output -raw` de `novedades_api_key_webhook` en preprod/datos o de
`api_key_webhook` en prod/novedades) y se entrega al proveedor por un canal seguro.

## Contrato del callback — CONFIRMADO (2026-10-19)

El proveedor hace **un POST por guía** a nuestra URL, con el header `x-api-key`, a medida que termina cada consulta (no manda un lote).
El cuerpo es un sobre; el resultado de la guía va dentro de `result`:

```json
{ "job_id": "…", "guia": "…", "ref": "…", "status": "ok", "result": { "guia": "…", "consultado_en": "…", "encontrado": true, "ultimo_motivo": "…", "flujo": [ … ], "motivos": [ … ], "remitente": { … }, "destinatario": { … } }, "error": null }
```

- `ref` vuelve tal cual lo enviamos. `status` distinto de `ok` indica que **no pudo consultar** la guía (ver abajo).
- Todo lo que depende de esta forma está en `Aplicacion/DTO/CallbackWsInterParser.cs`.
- Hecho verificado: la primera versión asumía el resultado plano (sin sobre) y dejó las 10 guías de prueba como `NoEncontrada` sin motivo; se corrigió el 2026-10-19 (pruebas de regresión en `CallbackConSobreTests.cs`). Con las 10 respuestas reales, el motivo extraído coincide en 10 de 10 con el `ultimo_motivo` del proveedor.
- **Privacidad:** `result` trae datos personales (`remitente`, `destinatario`) y comerciales (`envio`, `liquidacion`). Solo se guardan los campos de una **lista blanca** (`guia`, `consultado_en`, `encontrado`, `estado_actual`, `ultimo_motivo`, `flujo`, `motivos`).
- `estado_actual` del proveedor no es fiable (llega `Envío Admitido` aunque la guía ya esté devuelta); no se usa.

### Errores del proveedor

Si no pudo consultar la guía: `status` distinto de `ok` y/o `result: null`, con el motivo en `error` (texto u objeto con `message`). La solicitud queda `Fallida` con ese texto,
se cuenta en la métrica `CallbacksConErrorProveedor` y la guía se reintenta cuando venza el refresco. No se confunde con `encontrado:false`.

### Pendiente de confirmar con el proveedor

1. Reintentos ante `5xx`/timeout de nuestro webhook (pedimos espera creciente; sin reintento ante `4xx`).
2. Límites de tasa y tiempo esperado de procesamiento (nuestro plan de uso: 20 peticiones/s, ráfaga 40).
3. Que `x-api-key` sea **obligatoria** en su `POST /webhook/flujo-guia` (en su OpenAPI figura como opcional).
4. Un dominio propio y estable en lugar de `duckdns.org` (DNS dinámico gratuito: riesgo de disponibilidad).
5. La forma exacta del campo `error` cuando falla una consulta (hoy se acepta texto u objeto con `message`/`detail`/`mensaje`).

## Respuestas de nuestro webhook

Ver la tabla de códigos en [ApiLambdaNovedadesInter](../ApiLambdaNovedadesInter.md#response). Regla para el proveedor:
`200` = recibido; `5xx`/timeout = reintentar; `4xx` = no reintentar.

## Riesgos conocidos

- El servicio hace *scraping* de la web de Inter: puede romperse sin aviso. Lo cubren las alertas
  ([observabilidad](observabilidad.md)). Conviene validar con negocio que su uso es aceptable frente a Inter.
- Si el callback nunca llega, las guías quedan `Enviada` → tras 3 envíos `Fallida` y alerta `SinRespuesta`.
