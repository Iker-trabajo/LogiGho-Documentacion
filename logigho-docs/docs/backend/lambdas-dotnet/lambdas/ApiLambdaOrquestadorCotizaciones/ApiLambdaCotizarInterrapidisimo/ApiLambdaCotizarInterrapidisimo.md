## Autor: Iker Acevedo

Fecha creación: 2026-07-16
Estado: produccion

---

## Lambda: ApiLambdaCotizarInterrapidisimo

**Trigger:** Invocación Lambda-a-Lambda (AWS SDK) desde `ApiLambdaOrquestadorCotizaciones`

---

## ¿Qué hace?

Consulta el **cotizador de Interrapidísimo** y devuelve el flete real del envío. Es un *worker*: no aplica lógica de negocio de LogiGho, solo traduce el request, llama a la API de Inter y normaliza la respuesta.

El orquestador la invoca para obtener el **flete real** y sobre ese valor aplica la tarifa LogiGho.

---

## Accionador

| Método | Ruta | Auth |
| ------ | ---- | ---- |
| *(SDK)* | Invocación directa `RequestResponse` | Rol de ejecución del orquestador con `lambda:InvokeFunction` |
| `POST` | *(handler API Gateway disponible, no usado por el cotizador)* | — |

**Handler:** `ApiLambdaCotizarInterrapidisimo::ApiLambdaCotizarInterrapidisimo.Lambda.Handlers.DirectFunction::HandleAsync`

**Env var que lo referencia:** `FN_INTER_COTIZAR` (en el orquestador)

---

## Request

```json
{
  "valorContraPago": 100000,
  "idFormaPago": 1,
  "idLocalidadOrigen": "11001000",
  "idLocalidadDestino": "05001000",
  "peso": 3,
  "fecha": "2026-07-16T00:00:00Z"
}
```

| Campo | Tipo | Requerido | Descripción |
| ----- | ---- | --------- | ----------- |
| `valorContraPago` | `decimal` | Sí | Valor declarado / a recaudar. No puede ser negativo. |
| `idFormaPago` | `int` | Sí | Código de forma de pago de Inter. Debe ser `> 0`. |
| `idLocalidadOrigen` | `string` | Sí | Ciudad origen en **código DANE 8**. |
| `idLocalidadDestino` | `string` | Sí | Ciudad destino en **código DANE 8**. |
| `peso` | `decimal` | Sí | Peso en kilos. Debe ser `> 0`. Se **redondea hacia arriba** (`Math.Ceiling`) al llamar a Inter. |
| `fecha` | `datetime` | Sí | Fecha del envío. Se envía a Inter con formato `dd-MM-yyyy`. |

> El orquestador arma este request con `FromCommonMapper.ToInter(Common)`.

---

## Response

### Exitoso

Devuelve un **array** (Inter puede retornar varios servicios):

```json
[
  {
    "valorEnvio": 19180.00,
    "valorTotal": 19180.00,
    "valorPrimaSeguro": 1000.00,
    "valorContraPago": 0.0,
    "fechaEntrega": "2026-07-16T18:00:00",
    "valorKiloInicial": 12320.00,
    "valorKiloAdicional": 3430.00,
    "informacionEntrega": { "fechaEntrega": "2026-07-16T18:00:00", "tiempoEntrega": 1 }
  }
]
```

| Campo | Descripción |
| ----- | ----------- |
| `valorEnvio` | Flete que cobra Inter (= `Precio.Valor` de su API) |
| `valorTotal` | Mismo valor que `valorEnvio` (el orquestador lo **sobrescribe** con la tarifa LogiGho) |
| `valorPrimaSeguro` | Prima de seguro de Inter |
| `valorContraPago` | Valor de contrapago |
| `fechaEntrega` / `informacionEntrega` | Fecha y días de entrega |
| `valorKiloInicial` / `valorKiloAdicional` | Tarifas por kilo que reporta Inter |

Si Inter no devuelve resultados, retorna un **array vacío**.

### Errores

| Código | Cuándo |
| ------ | ------ |
| `ArgumentException` | Falta `idLocalidadOrigen`/`idLocalidadDestino`, `peso <= 0`, `idFormaPago <= 0` o `valorContraPago < 0` |
| `HttpRequestException` | Inter respondió con código != 2xx (incluye el detalle) |
| `InvalidOperationException` | Falta alguna env var (`INTER_CLIENT_ID`, `INTER_APP_SIGNATURE`, `INTER_SECURITY_TOKEN`) |

> Al invocarse por SDK, los errores llegan al orquestador como `FunctionError=Unhandled` y este los devuelve en `interrapidisimo.error`.

---

### Qué hace cada capa

| Capa | Archivo | Responsabilidad |
| ---- | ------- | --------------- |
| **Lambda/Handlers** | `DirectFunction.cs` | Punto de entrada para invocación por SDK (objeto directo). |
| **Lambda/Handlers** | `ApiGatewayFunction.cs` | Punto de entrada HTTP (disponible, no usado por el cotizador). |
| **Lambda** | `CompositionRoot.cs` | Composición de dependencias (servicio + `HttpClient`). |
| **Lambda** | `Cors.cs`, `Serialization.cs` | Headers CORS y configuración del serializador. |
| **Aplicacion** | `InterCotizadorService.cs` | **Valida** el request (reglas de negocio de entrada) y delega al cliente HTTP. |
| **Dominio/Interfaces** | `IInterCotizadorService.cs` | Contrato del servicio. |
| **Dominio/Entidades** | `CotizarInterRequest.cs` | Modelo del request. |
| **Models** | `CotizarInterRespuesta.cs` | Modelo de la respuesta normalizada. |
| **Infraestructura** | `InterPricingApiClient.cs` | Cliente HTTP de Inter: arma la URL, pone los headers, mapea la respuesta. |

---

## Dependencias externas

| Servicio | Uso |
| -------- | --- |
| `API Interrapidísimo` | Cotizador: `https://www3.interrapidisimo.com/ApiServInter/api/CotizadorCliente/ResultadoListaCotizar` |

### Variables de entorno

| Variable | Requerida | Uso |
| -------- | --------- | --- |
| `INTER_BASE_URL` | No | Endpoint del cotizador. Tiene default si no se configura. |
| `INTER_CLIENT_ID` | **Sí** | Id de cliente de Inter (va en la URL). Lanza excepción si falta. |
| `INTER_APP_SIGNATURE` | **Sí** | Header `x-app-signature`. Lanza excepción si falta. |
| `INTER_SECURITY_TOKEN` | **Sí** | Header `x-app-security_token`. Lanza excepción si falta. |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|-------|-------|--------|
| — | — | Sin cambios en esta iteración. La lógica de trayectos/kilo adicional de Inter vive en el **orquestador**, no aquí. |

---

## Observaciones

Sin observaciones.
