## Autor: Iker Acevedo

Fecha creación: 2026-07-16
Estado: desarrollo (⚠️ desplegada pero NO se invoca — ver Deuda técnica)

---

## Lambda: ApiLambdaCotizarEnvia

**Trigger:** Invocación Lambda-a-Lambda (AWS SDK) — *actualmente **no se invoca***
**AOT:** No

---

## ¿Qué hace?

Consulta el servicio de **liquidación de Envía** (`hub.envia.co`) y devuelve el flete real del envío. Es un *worker*, igual que los de Inter y Servientrega.

> ⚠️ **Hoy el cotizador NO la usa.** El orquestador estima la tarifa de Envía de forma **geográfica** porque no se logró consumir la API de Envía de forma confiable. La lambda sigue desplegada y funcional a nivel de código, esperando la reunión con la transportadora. Ver **Deuda técnica** abajo.

---

## Accionador

| Método | Ruta | Auth |
| ------ | ---- | ---- |
| *(SDK)* | Invocación directa `RequestResponse` | Rol de ejecución del orquestador con `lambda:InvokeFunction` |
| `POST` | *(handler API Gateway disponible)* | — |

**Handler:** `ApiLambdaCotizarEnvia::ApiLambdaCotizarEnvia.Lambda.Handlers.DirectFunction::HandleAsync`

**Env var que lo referencia:** `FN_ENVIA_LIQUIDACION` (en el orquestador) — **actualmente sin uso**

---

## Request

```json
{
  "ciudad_origen": "11001000",
  "ciudad_destino": "05001000",
  "cod_formapago": "6",
  "cod_servicio": 12,
  "mca_docinternacional": 0,
  "info_contenido": { "num_documentos": "12345-67890" },
  "info_cubicacion": [
    { "declarado": 100000, "peso": 3, "alto": 10, "ancho": 10, "largo": 10, "cantidad": "1" }
  ]
}
```

| Campo | Tipo | Requerido | Descripción |
| ----- | ---- | --------- | ----------- |
| `ciudad_origen` | `string` | Sí | Ciudad origen en **código DANE 8**. |
| `ciudad_destino` | `string` | Sí | Ciudad destino en **código DANE 8**. |
| `cod_formapago` | `string` | Sí | Forma de pago: `4` = Crédito · `6` = Contado · `7` = Contraentrega. |
| `cod_servicio` | `int` | Sí | Modalidad. `12` = Paquete Terrestre (1–8 kg, una unidad). |
| `mca_docinternacional` | `int` | Sí | `0` si no aplica. |
| `info_contenido.num_documentos` | `string` | Sí | Número de factura/documento. |
| `info_cubicacion[]` | `array` | Sí | Peso, dimensiones, valor declarado y cantidad. Reemplaza a `num_unidades`/`mpesoreal_k`/`valor_declarado`. |

> El orquestador armaría este request con `FromCommonMapper.ToEnvia(Common)`.

---

## Response

### Exitoso

```json
{
  "respuesta": "",
  "k_cobrados": 1,
  "valor_flete": 3900,
  "valor_costom": 0,
  "valor_otros": 0,
  "dias_entrega": 1,
  "guia": null,
  "urlguia": null,
  "cod_postaldestino": "111311026"
}
```

| Campo | Descripción |
| ----- | ----------- |
| `respuesta` | **Vacío = éxito.** Si trae texto, es el mensaje de error. |
| `valor_flete` | **Flete real** que cobra Envía (el que usaría el orquestador). |
| `k_cobrados` | El mayor entre peso real y volumen. |
| `valor_costom` | Valor adicional según valor declarado. |
| `dias_entrega` | Tiempo ofrecido. |
| `guia` / `urlguia` | Solo en generación de guía, no en liquidación. |

⚠️ **Envía responde `200 OK` incluso con errores de negocio**: el error viene en el campo `respuesta` y los valores en `0`. Hay que revisar `respuesta`, no solo el status code.

### Errores

| Código | Cuándo |
| ------ | ------ |
| `InvalidOperationException` | Falta la env var `ENVIA_ENDPOINT` |
| `HttpRequestException` | Envía respondió != 2xx (incluye el detalle) |
| `TimeoutException` | Se agotó el presupuesto de tiempo (ver `Timeouts`) |

---


## Qué hace cada capa

| Capa | Archivo | Responsabilidad |
| ---- | ------- | --------------- |
| **Lambda/Handlers** | `DirectFunction.cs` | Punto de entrada para invocación por SDK (objeto directo). |
| **Lambda/Handlers** | `ApiGatewayFunction.cs` | Punto de entrada HTTP. |
| **Lambda** | `CompositionRoot.cs` | Composición del servicio + `HttpClient`. |
| **Lambda** | `Timeouts.cs` | **Presupuesto de tiempo inteligente**: respeta el `RemainingTime` del contexto Lambda (menos 1s de seguridad) para no morir por timeout de la lambda. |
| **Lambda** | `Cors.cs`, `Serialization.cs` | Headers CORS y serializador. |
| **Aplicacion/Services** | `IEnviaLiquidacionService.cs` | Contrato del servicio. |
| **Aplicacion/Services** | `EnviaLiquidacionService.cs` | Delega al cliente HTTP (capa fina). |
| **Dominio/Entidades** | `EnviaLiquidacionRequest.cs` | Request + `InfoContenido` + `InfoCubicacion`. |
| **Dominio/Entidades** | `EnviaLiquidacionResponse.cs` | Respuesta de Envía. |
| **Infraestructura/Http** | `EnviaLiquidacionApiClient.cs` | Cliente HTTP: POST JSON, fuerza HTTP/1.1, convierte cancelaciones en `TimeoutException` con diagnóstico. |

---

## Dependencias externas

| Servicio | Uso |
| -------- | --- |
| `API Envía` | Liquidación (cotización). Prod: `https://hub.envia.co/ServicioLiquidacionREST/Service1.svc/Liquidacion/` · Pruebas: `.../ServicioLiquidacionRESTpruebas/...` |

### Variables de entorno

| Variable | Requerida | Uso |
| -------- | --------- | --- |
| `ENVIA_ENDPOINT` | **Sí** | Endpoint de liquidación. **No tiene default**: lanza excepción si falta. Se configura directamente en la lambda (no está en el `aws-lambda-tools-defaults.json`). |
| `REQUEST_TIMEOUT_SECONDS` | No | Presupuesto total de la petición (default `20`). |
| `HTTP_TIMEOUT_SECONDS` | No | Timeout del `HttpClient` (default `25`). |
| `USE_REMAININGTIME_CAP` | No | Si respeta el `RemainingTime` del contexto (default `true`). |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|-------|-------|--------|
| — | — | Sin cambios en esta iteración. Se evaluó integrarla al cotizador pero se dejó como deuda técnica (ver abajo). |

---

## Observaciones

### Deuda técnica: Envía no está integrada al cotizador

**Estado:** el orquestador **NO invoca** esta lambda. Estima la tarifa de Envía geográficamente.

**Por qué:** el endpoint de liquidación rechaza sistemáticamente con:
> `"Ciudad sin cubrimiento o valor producto supera el valor maximo para proceso recaudos."`

