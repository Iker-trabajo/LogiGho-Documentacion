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

`POST /webhooks/ws-inter/flujo-guia` en el API Gateway REST propio del dominio:

| Entorno | API | Stage |
| ------- | --- | ----- |
| PreProd | `Webhooks-LogiGho-PreProd` (regional) | `preprod` |
| Producción | `Webhooks-LogiGho-Prod` (regional) | `produccion` |

La URL exacta es la salida `url_webhook` de Terraform (`terraform output url_webhook`).

**Seguridad.** El método exige API key (`x-api-key`): sin ella API Gateway responde `403 {"message":"Forbidden"}` sin invocar
la lambda. La lambda la vuelve a comprobar (comparación en tiempo constante). El *usage plan* limita a 20 peticiones/s
(ráfaga 40) y 50 000 al día. La clave la genera Terraform (`terraform output -raw api_key_webhook`) y se entrega al
proveedor por un canal seguro.

## Contrato del callback — PROVISIONAL

> ws-inter aún no documenta el callback. Se asume que envía el mismo objeto `FlujoGuiaResponse` de su OpenAPI, más
> `job_id` y el `ref` que le enviamos, como **un objeto** o un **arreglo de hasta 100**. Todo lo que depende de esto
> está en `Aplicacion/DTO/CallbackWsInterParser.cs`.

Lo que se le pidió al proveedor (pendiente de respuesta):

1. Forma exacta del callback y una muestra real.
2. URL de callback fija por entorno (preprod y producción) y header `x-api-key` en cada llamada.
3. Reintentos con espera creciente ante `5xx`/timeout; sin reintento ante `4xx`.
4. Límites de tasa, tiempo esperado de procesamiento y si aceptan varias guías en una llamada.
5. Que `x-api-key` sea **obligatoria** en su `POST /webhook/flujo-guia` (hoy figura como opcional).
6. Un dominio propio y estable en lugar de `duckdns.org` (DNS dinámico gratuito: riesgo de disponibilidad).
7. Qué devuelven cuando la guía no existe (`encontrado:false`) o si repetimos el mismo `ref`.

## Respuestas de nuestro webhook

Ver la tabla de códigos en [ApiLambdaNovedadesInter](../ApiLambdaNovedadesInter.md#response). Regla para el proveedor:
`200` = recibido; `5xx`/timeout = reintentar; `4xx` = no reintentar.

## Riesgos conocidos

- El servicio hace *scraping* de la web de Inter: puede romperse sin aviso. Lo cubren las alertas
  ([observabilidad](observabilidad.md)). Conviene validar con negocio que su uso es aceptable frente a Inter.
- Si el callback nunca llega, las guías quedan `Enviada` → tras 3 envíos `Fallida` y alerta `SinRespuesta`.
