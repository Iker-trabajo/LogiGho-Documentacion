## Autor: Iker Acevedo

Fecha creación: 2026-07-16

Estado: produccion

---

## Lambda: ApiLambdaCotizarInterrapidisimo

**Trigger:** Invocación Lambda-a-Lambda (AWS SDK) desde `ApiLambdaOrquestadorCotizaciones`

---

## ¿Qué hace?

Consulta el servicio de **cotizador de Interrapidísimo** y devuelve el flete real del envío. Es un *worker*: no aplica lógica de negocio de LogiGho, solo traduce el request, llama a la API de Inter y normaliza la respuesta.

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
  "valorDeclarado": 100000,
  "aplicaContrapago": true,
  "idLocalidadOrigen": "11001000",
  "idLocalidadDestino": "05001000",
  "peso": 3,
  "fecha": "2026-07-16T00:00:00Z"
}
```

| Campo | Tipo | Requerido | Descripción |
| ----- | ---- | --------- | ----------- |
| `valorDeclarado` | `decimal` | Sí | Valor de la mercancía. Sirve **tanto para el seguro como para el monto a recaudar** — a diferencia de Envía/Servientrega, Inter no separa ambos conceptos. **Siempre es el valor real del pedido**, con o sin recaudo (no hay valor mínimo fijo). |
| `aplicaContrapago` | `bool` | Sí | El switch con/sin recaudo (`Common.ConRecaudo`). `true` = con recaudo, `false` = sin recaudo. |
| `idLocalidadOrigen` | `string` | Sí | Ciudad origen en **código DANE 8**. |
| `idLocalidadDestino` | `string` | Sí | Ciudad destino en **código DANE 8**. |
| `peso` | `decimal` | Sí | Peso en kilos. Debe ser `> 0`. Se **redondea hacia arriba** (`Math.Ceiling`) al llamar a Inter. |
| `fecha` | `datetime` | Sí | Fecha del envío. Se envía a Inter con formato `dd-MM-yyyy`. |

> El orquestador arma este request con `FromCommonMapper.ToInter(Common)`.
>
> ⚠️ **Campos que cambiaron (2026-09-09)**: `valorContraPago` se separó en `valorDeclarado` (que va a Inter como valor declarado real) porque antes el cliente HTTP mandaba `ValorContraPago` en el campo que debía llevar `ValorDeclarado` — un bug de mapeo. `idFormaPago` se **eliminó**: la doc de Inter confirma que "no aplica para cliente crédito integrado", era un campo muerto.

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
| `ArgumentException` | Falta `idLocalidadOrigen`/`idLocalidadDestino`, `peso <= 0` o `valorDeclarado < 0` |
| `HttpRequestException` | Inter respondió con código != 2xx (incluye el detalle) |
| `InvalidOperationException` | Falta alguna env var (`INTER_CLIENT_ID`, `INTER_APP_SIGNATURE`, `INTER_SECURITY_TOKEN`) |

> Al invocarse por SDK, los errores llegan al orquestador como `FunctionError=Unhandled` y este los devuelve en `interrapidisimo.error`.

---

## Cómo se consume la API real de Interrapidísimo

> 💡 Guía rápida para cuando toque volver a tocar esto sin tener el contexto fresco.

**Cliente:** `InterPricingApiClient.CotizarAsync` (`Infraestructura/InterPricingApiClient.cs`)

| | |
| --- | --- |
| **Método HTTP** | `GET` |
| **URL base** | `https://www3.interrapidisimo.com/ApiServInter/api/CotizadorCliente/ResultadoListaCotizar` (o `INTER_BASE_URL` si está seteada) |
| **Forma de la URL** | Todo va **en el path, no en query ni body** — es una API tipo RPC por segmentos: `{baseUrl}/{clienteId}/{idLocalidadOrigen}/{idLocalidadDestino}/{peso}/{valorDeclarado}/{tipoEntrega}/{fecha}/{aplicaContrapago}` |
| **Autenticación** | 2 headers fijos, **no hay token que expire ni login previo**: `x-app-signature: {INTER_APP_SIGNATURE}` y `x-app-security_token: {INTER_SECURITY_TOKEN}` |

### Cómo se arma cada segmento de la URL

| Segmento | De dónde sale |
| -------- | -------------- |
| `{clienteId}` | Env var `INTER_CLIENT_ID` — identifica la cuenta de LogiGho ante Inter, va en la URL (no es un header). |
| `{idLocalidadOrigen}` / `{idLocalidadDestino}` | Código DANE de 8 dígitos, tal cual llega en el request. |
| `{peso}` | `Math.Ceiling(payload.Peso)` — Inter espera el peso **redondeado hacia arriba**, sin decimales. |
| `{valorDeclarado}` | Valor real del pedido (con o sin recaudo, siempre el mismo). |
| `{tipoEntrega}` | **Constante `1`** — no configurable, hardcodeado en el cliente. |
| `{fecha}` | Formateada `dd-MM-yyyy` (no ISO). |
| `{aplicaContrapago}` | `"TRUE"` / `"FALSE"` (string, mayúsculas) — **este es el switch de con/sin recaudo**. |

### Con / sin recaudo en la práctica

- **Con recaudo:** `aplicaContrapago=TRUE` al final de la URL.
- **Sin recaudo:** `aplicaContrapago=FALSE` al final de la URL.
- `valorDeclarado` **no cambia** entre los dos casos — es siempre el valor real del pedido (ver Observaciones abajo, se descartó un piso fijo).

### Si hay que probar esto a mano (Postman, curl)

1. Conseguir `INTER_CLIENT_ID`, `INTER_APP_SIGNATURE`, `INTER_SECURITY_TOKEN` (no van en este repo de docs — pedirlos a quien tenga acceso al `.env`/Secrets Manager).
2. Armar la URL completa reemplazando cada segmento en el orden de la tabla de arriba.
3. `GET` con los 2 headers. Respuesta esperada: un array JSON (puede venir vacío si no hay servicios disponibles para esa ruta/peso).
4. Si Inter responde algo distinto de 2xx, el body trae el detalle del error — el cliente lo propaga en el mensaje de la excepción.

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
| 2026-09-09 | Iker Acevedo | Soporte con/sin recaudo (`aplicaContrapago`). Bug corregido: se separó `valorDeclarado` de `valorContraPago` (el cliente HTTP mandaba el campo equivocado). Eliminado `idFormaPago` (campo muerto, no aplica para cliente crédito integrado según la doc de Inter). Migrada al patrón Strategy (`CotizadorInter`). |
| — | — | Sin cambios previos. La lógica de trayectos/kilo adicional de Inter vive en el **orquestador**, no aquí. |

---

## Observaciones

- Sin observaciones
