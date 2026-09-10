## Autor: Iker Acevedo

Fecha creación: 2026-07-16

Última actualización: 2026-09-09

Estado: produccion

---

## Lambda: ApiLambdaCotizarEnvia

**Trigger:** Invocación Lambda-a-Lambda (AWS SDK) desde `ApiLambdaOrquestadorCotizaciones`
**AOT:** No

---

## ¿Qué hace?

Consulta el servicio de **liquidación de Envía** (`hub.envia.co`) y devuelve el flete real del envío. Es un *worker*, igual que los de Inter y Servientrega.

**Desde el 2026-09-09 el orquestador SÍ la invoca de verdad.** Antes se estimaba la tarifa de Envía de forma geográfica porque no se lograba consumir la API de Envía de forma confiable — la causa raíz era un bug de negocio (ver **Observaciones**), no un problema de la lambda en sí.

---

## Accionador

| Método | Ruta | Auth |
| ------ | ---- | ---- |
| *(SDK)* | Invocación directa `RequestResponse` | Rol de ejecución del orquestador con `lambda:InvokeFunction` |
| `POST` | *(handler API Gateway disponible)* | — |

**Handler:** `ApiLambdaCotizarEnvia::ApiLambdaCotizarEnvia.Lambda.Handlers.DirectFunction::HandleAsync`

**Env var que lo referencia:** `FN_ENVIA_LIQUIDACION` (en el orquestador)

---

## Request

```json
{
  "ciudad_origen": "11001000",
  "ciudad_destino": "05001000",
  "cod_formapago": "6",
  "cod_servicio": 12,
  "mca_docinternacional": 0,
  "cod_regional_cta": "11",
  "cod_oficina_cta": "110010",
  "cod_cuenta": "XXXXXX",
  "con_cartaporte": "0",
  "info_contenido": { "num_documentos": "12345-67890", "valorproducto": "100000" },
  "info_cubicacion": [
    { "declarado": 100000, "peso": 3, "alto": 10, "ancho": 10, "largo": 10, "cantidad": 1 }
  ]
}
```

| Campo | Tipo | Requerido | Descripción |
| ----- | ---- | --------- | ----------- |
| `ciudad_origen` | `string` | Sí | Ciudad origen en **código DANE 8**. |
| `ciudad_destino` | `string` | Sí | Ciudad destino en **código DANE 8**. |
| `cod_formapago` | `string` | Sí | Forma de pago: `4` = Crédito (**con recaudo** en Envía) · `6` = Contado · `7` = Contraentrega (**sin recaudo** en Envía). Ver nota de semántica invertida abajo. |
| `cod_servicio` | `int` | Sí | Modalidad. `12` = Paquete Terrestre (1–8 kg, una unidad). |
| `mca_docinternacional` | `int` | Sí | `0` si no aplica. |
| `cod_regional_cta` / `cod_oficina_cta` / `cod_cuenta` | `string` | Sí | Datos de la cuenta de Envía a usar. El orquestador los resuelve con `EnviaCuentaResolver` según `ConRecaudo` — ver [ADR-001](../ApiLambdaOrquestadorCotizaciones/ADR-001-strategy-cotizadores.md). |
| `con_cartaporte` | `string` | Sí | `"0"`/`"1"` (string, no bool/int). Requerido por el contrato de Envía. |
| `info_contenido.num_documentos` | `string` | Sí | Número de factura/documento. |
| `info_contenido.valorproducto` | `string` | Sí | Valor de la mercancía como **string** (así lo espera Envía), para el cálculo de `valor_costom`. Solo aplica de verdad en cuentas con recaudo — sin recaudo se manda `"0"`. |
| `info_cubicacion[]` | `array` | Sí | Peso, dimensiones, valor declarado y cantidad. Reemplaza a `num_unidades`/`mpesoreal_k`/`valor_declarado`. |

> ⚠️ **Semántica invertida respecto al resto del mercado**: en Envía, Crédito (`4`) = CON recaudo y Contraentrega (`7`) = SIN recaudo — al revés de Inter y Servientrega, donde con recaudo = contado/contraentrega. Ver el ADR del orquestador para la tabla completa.
>
> El orquestador arma este request con `FromCommonMapper.ToEnvia(Common)` + `EnviaCuentaResolver`.

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

⚠️ **Envía responde `200 OK` incluso con errores de negocio**: el error viene en el campo `respuesta` y los valores en `0`. Hay que revisar `respuesta`, no solo el status code. Esto es exactamente lo que antes hacía que el orquestador leyera un error (ej. peso fuera de rango) como una cotización válida con flete `$0`.

### Errores

| Código | Cuándo |
| ------ | ------ |
| `EnviaLiquidacionException` | Envía respondió `200` pero `respuesta != ""` (error de negocio: peso fuera de rango, ciudad sin cubrimiento, etc.). |
| `InvalidOperationException` | Falta la env var `ENVIA_ENDPOINT` |
| `HttpRequestException` | Envía respondió != 2xx (incluye el detalle) |
| `TimeoutException` | Se agotó el presupuesto de tiempo (ver `Timeouts`) |

---

## Cómo se consume la API real de Envía

> 💡 Guía rápida para cuando toque volver a tocar esto sin tener el contexto fresco.

**Cliente:** `EnviaLiquidacionApiClient.LiquidarAsync` (`Infraestructura/Http/EnviaLiquidacionApiClient.cs`)

| | |
| --- | --- |
| **Método HTTP** | `POST` |
| **URL** | Prod: `https://hub.envia.co/ServicioLiquidacionREST/Service1.svc/Liquidacion/` · Pruebas: `.../ServicioLiquidacionRESTpruebas/...` — siempre viene de la env var `ENVIA_ENDPOINT`, **sin default en código** (si falta, la lambda ni arranca). |
| **Autenticación** | ⚠️ **No hay token, API key ni Bearer.** La identidad de la cuenta va **dentro del body**: `cod_regional_cta` + `cod_oficina_cta` + `cod_cuenta`. Envía autentica por los datos de cuenta que llegan en cada request, no por un header aparte. |
| **Detalle de transporte** | Fuerza `HttpVersion.Version11` (`VersionPolicy: RequestVersionOrLower`) — Envía tuvo problemas históricos con HTTP/2, por eso el cliente lo baja a la fuerza. |

### De dónde salen los datos de cuenta

`cod_regional_cta` / `cod_oficina_cta` / `cod_cuenta` **no los arma esta lambda** — llegan ya resueltos en el request. Quien decide cuál cuenta usar es `EnviaCuentaResolver` en el orquestador, según `ConRecaudo` (ver [ADR-001](../ApiLambdaOrquestadorCotizaciones/ADR-001-strategy-cotizadores.md)). Si vas a probar esta lambda de forma aislada (sin pasar por el orquestador), tenés que armar esos 3 campos a mano con la cuenta correcta según el caso que quieras probar.

### Con / sin recaudo en la práctica

| | Con recaudo | Sin recaudo |
| --- | --- | --- |
| Cuenta (`cod_regional_cta`/`cod_oficina_cta`/`cod_cuenta`) | `ENVIA_CUENTA_RECAUDO` | `ENVIA_CUENTA_SIN_RECAUDO` |
| `cod_formapago` | `"4"` (Crédito) | `"7"` (Contraentrega) |
| `info_contenido.valorproducto` | Valor real del pedido | `"0"` |

⚠️ Recordar la inversión: en Envía "Crédito" es el código de **con** recaudo, al revés de Inter/Servientrega donde crédito = sin recaudo.

### Si hay que probar esto a mano (Postman, curl)

1. Conseguir el endpoint (prod o pruebas) y los 3 datos de cuenta según el caso (con o sin recaudo) — pedirlos a quien tenga acceso al Secrets Manager / configuración de Envía, no están en este repo de docs.
2. `POST` con el body completo (`ciudad_origen`, `ciudad_destino`, `cod_formapago`, `cod_servicio`, datos de cuenta, `info_contenido`, `info_cubicacion`).
3. **Revisar siempre el campo `respuesta` del body**, aunque el HTTP sea `200`. Si trae texto, es un error de negocio, no una cotización válida.
4. Si necesitas depurar el endpoint de pruebas vs producción, recordá que son **hosts distintos** (`ServicioLiquidacionREST` vs `ServicioLiquidacionRESTpruebas`), no solo un query param.

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
| **Aplicacion/Services** | `EnviaLiquidacionService.cs` | Delega al cliente HTTP (capa fina) y lanza `EnviaLiquidacionException` si `respuesta != ""`. |
| **Dominio/Entidades** | `EnviaLiquidacionRequest.cs` | Request + `InfoContenido` (con `valorproducto`) + `InfoCubicacion` + datos de cuenta (`cod_regional_cta`, `cod_oficina_cta`, `cod_cuenta`, `con_cartaporte`). |
| **Dominio/Entidades** | `EnviaLiquidacionResponse.cs` | Respuesta de Envía. |
| **Dominio/Excepciones** | `EnviaLiquidacionException.cs` | Se lanza cuando Envía responde `200` con `respuesta` lleno (error de negocio). Antes ese caso se leía como cotización válida con flete `$0`. |
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
| `ENVIA_CUENTA_RECAUDO` / `ENVIA_CUENTA_SIN_RECAUDO` | **Sí** *(en el orquestador)* | No son env vars de esta lambda — viven en `ApiLambdaOrquestadorCotizaciones` y `EnviaCuentaResolver` las usa para armar `cod_regional_cta`/`cod_oficina_cta`/`cod_cuenta` antes de invocar esta lambda. |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|-------|-------|--------|
| 2026-09-09 | Iker Acevedo | **Integrada al cotizador**: el orquestador ya la invoca de verdad. Se completó `EnviaLiquidacionRequest` con datos de cuenta (`cod_regional_cta`, `cod_oficina_cta`, `cod_cuenta`, `con_cartaporte`, `info_contenido.valorproducto`). Nueva `EnviaLiquidacionException` para el bug de error de negocio con HTTP 200. |
| 2026-07-16 | Iker Acevedo | Creación de la lambda. Se evaluó integrarla al cotizador pero se dejó como deuda técnica en ese momento. |

---

## Observaciones

### El bug que impedía integrarla (ya corregido)

El endpoint de liquidación de Envía rechazaba sistemáticamente con:
> `"Ciudad sin cubrimiento o valor producto supera el valor maximo para proceso recaudos."`

La causa real no era la ciudad ni el valor: **faltaban campos del contrato** (`cod_regional_cta`, `cod_oficina_cta`, `cod_cuenta`, `con_cartaporte`, `info_contenido.valorproducto`) y, además, el orquestador no distinguía el caso de error de negocio (HTTP 200 con `respuesta` lleno) de una cotización válida. Corregido en la Ronda 1 de `feature/integracion-cotizador-envia` (2026-09-09): request completado + `EnviaLiquidacionException`.

- **Guía con recaudo para Envía sin-recaudo** sigue sin soportarse operativamente (se puede crear el pedido pero no se factura bien) — fuera de alcance de esta feature.
- `cubrimiento`: esta lambda no lo expone en el response documentado aquí, pero el modelo de respuesta del orquestador sí lo trae; el front decidió no mostrarlo.

