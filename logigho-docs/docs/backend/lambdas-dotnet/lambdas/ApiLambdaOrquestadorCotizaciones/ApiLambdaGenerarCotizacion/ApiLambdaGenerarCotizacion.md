## Autor: Iker Acevedo

Fecha creación: 2026-07-16
Estado: produccion

---

## Lambda: ApiLambdaGenerarCotizacion (Servientrega)

**Trigger:** Invocación Lambda-a-Lambda (AWS SDK) desde `ApiLambdaOrquestadorCotizaciones` · 
**AOT:** No

---

## ¿Qué hace?

Consulta el **Cotizador Corporativo de Servientrega** y devuelve el flete real del envío. Es un *worker*: no aplica lógica de negocio de LogiGho.

**Autogestiona su autenticación**: si el request no trae `Token`, hace login contra Servientrega con las credenciales de sus variables de entorno y usa el token resultante para cotizar. Por eso el orquestador solo necesita invocarla **una vez**, sin preocuparse por el token.

---

## Accionador

| Método | Ruta | Auth |
| ------ | ---- | ---- |
| *(SDK)* | Invocación directa `RequestResponse` | Rol de ejecución del orquestador con `lambda:InvokeFunction` |
| `POST` | *(handler API Gateway disponible, no usado por el cotizador)* | — |

**Handler:** `ApiLambdaGenerarCotizacion::ApiLambdaGenerarCotizacion.Lambda.Handlers.DirectFunction::HandleAsync`

**Env var que lo referencia:** `FN_GENERAR_COTIZACION` (en el orquestador)

---

## Request

```json
{
  "IdProducto": 3,
  "NumeroPiezas": 1,
  "Piezas": [{ "Peso": 3, "Largo": 10, "Ancho": 10, "Alto": 10 }],
  "ValorDeclarado": 100000,
  "IdDaneCiudadOrigen": "11001000",
  "IdDaneCiudadDestino": "05001000",
  "EnvioConCobro": true,
  "FormaPago": 1,
  "TiempoEntrega": 3,
  "MedioTransporte": 1,
  "NumRecaudo": 2,
  "Token": ""
}
```

| Campo | Tipo | Requerido | Descripción |
| ----- | ---- | --------- | ----------- |
| `Piezas[]` | `array` | Sí | Peso y dimensiones. **Es lo que realmente se envía** a Servientrega. |
| `ValorDeclarado` | `int` | Sí | Valor de la mercancía. |
| `IdDaneCiudadOrigen` | `string` | Sí | Ciudad origen en **código DANE**. |
| `IdDaneCiudadDestino` | `string` | Sí | Ciudad destino en **código DANE**. |
| `EnvioConCobro` | `bool` | Sí | Si el envío lleva logística de cobro. |
| `Token` | `string` | **No** | Token de Servientrega. **Si viene vacío, la lambda lo genera sola.** |
| `IdProducto`, `NumeroPiezas`, `FormaPago`, `TiempoEntrega`, `MedioTransporte`, `NumRecaudo` | `int` | Sí | Se reciben, pero ⚠️ **el servicio los sobrescribe** con valores fijos (ver Observaciones). |

> El orquestador arma este request con `FromCommonMapper.ToServientrega(Common)` y **no envía Token** (por eso la lambda lo genera).

---

## Response

### Exitoso

```json
{
  "ValorFlete": 15500.00,
  "ValorSobreFlete": 3000.00,
  "ValorTotal": 18500.00,
  "Informacion": null
}
```

| Campo | Descripción |
| ----- | ----------- |
| `ValorFlete` | **Flete real** que cobra Servientrega. Es el que usa el orquestador para la tarifa LogiGho. |
| `ValorSobreFlete` | Sobreflete (seguro que cobra Servientrega). **No se usa** en la fórmula LogiGho (LogiGho aplica su propio seguro). |
| `ValorTotal` | `ValorFlete + ValorSobreFlete` |

### Errores

| Código | Cuándo |
| ------ | ------ |
| `InvalidOperationException` | Falta `SERVI_LOGIN`, `SERVI_PASSWORD` o `SERVI_COD_FACTURACION` |
| `InvalidOperationException` | Servientrega respondió el login con `estado: false` o token vacío |
| `HttpRequestException` | Servientrega respondió != 2xx (login o cotización) |

> Al invocarse por SDK, los errores llegan al orquestador como `FunctionError=Unhandled` y este los devuelve en `servientrega.error`.

---

## Qué hace cada capa

| Capa | Archivo | Responsabilidad |
| ---- | ------- | --------------- |
| **Lambda/Handlers** | `DirectFunction.cs` | Punto de entrada para invocación por SDK (objeto directo). |
| **Lambda/Handlers** | `ApiGatewayFunction.cs` | Punto de entrada HTTP (disponible, no usado por el cotizador). |
| **Lambda** | `CompositionRoot.cs` | Composición del servicio (`Lazy<ICotizacionService>`). |
| **Lambda** | `Cors.cs`, `Serialization.cs` | Headers CORS y serializador. |
| **Infraestructura/Servicios** | `ICotizacionService.cs` | Contrato del servicio. |
| **Infraestructura/Servicios** | `CotizacionService.cs` | **Todo el trabajo**: genera el token si hace falta, arma el request y llama a Servientrega. |
| **Dominio/Entidades** | `GenerarCotizacionRequest.cs` | Request de entrada + `GenerarCotizacionSTRequest` (el que se manda a Servientrega) + `Pieza`. |
| **Dominio/Entidades** | `GenerarCotizacionResponse.cs` | Respuesta de la cotización. |
| **Dominio/Entidades** | `ServientregaTokenResponse.cs` | Respuesta del login (`estado`, `token`). |

### Cómo funciona el token (autenticación)

Según la doc de Servientrega (*Cotizador Corporativo*):

- **Endpoint:** `POST /api/Autenticacion/Login`
- **Body:** `{ "login", "password", "codFacturacion" }`
- **La `password` NO es texto plano**: es la **contraseña cifrada de Sisclinet**.
- **Respuesta:** `{ nombre, login, codFacturacion, idCliente, estado, token, expiration }`

La lambda valida `estado == true` y `token` no vacío antes de continuar (**fail-loud**): Servientrega puede responder `200 OK` con `estado: false` y token vacío, y `EnsureSuccessStatusCode()` no lo detectaría.

---

## Dependencias externas

| Servicio | Uso |
| -------- | --- |
| `API Servientrega` | Login: `/api/Autenticacion/Login` · Cotización: `/api/Cotizacion` |

### Variables de entorno

| Variable | Requerida | Uso |
| -------- | --------- | --- |
| `SERVIENTREGA_ENDPOINT` | No | Endpoint de cotización. Tiene default. |
| `SERVIENTREGA_AUTH_ENDPOINT` | No | Endpoint del login. Tiene default. |
| `SERVI_LOGIN` | **Sí** | Usuario de Servientrega (Sisclinet). |
| `SERVI_PASSWORD` | **Sí** | Contraseña **cifrada** de Sisclinet (no texto plano). |
| `SERVI_COD_FACTURACION` | **Sí** | Código de facturación del cliente. |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|-------|-------|--------|
| 2026-07-16 | Iker Acevedo | **La lambda genera su propio token**: si el request no trae `Token`, hace login con `SERVI_LOGIN`/`SERVI_PASSWORD`/`SERVI_COD_FACTURACION` |
| 2026-07-16 | Iker Acevedo | Nuevo modelo `ServientregaTokenResponse` + validación de `estado`/`token` (fail-loud) |
| 2026-07-16 | Iker Acevedo | Nuevas env vars `SERVI_LOGIN`, `SERVI_PASSWORD`, `SERVI_COD_FACTURACION` |
| 2026-07-16 | Iker Acevedo | `aws-lambda-tools-defaults.json`: `configuration` de `Debug` → `Release` |

---

## Observaciones

Sin observaciones.
