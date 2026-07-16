## Autor: Iker Acevedo

Fecha creación: 2026-07-16

Estado: produccion

---

## Lambda: ApiLambdaOrquestadorCotizaciones

**Trigger:** API Gateway

**AOT:** No

---

## ¿Qué hace?

Cotiza un mismo envío en las **3 transportadoras** (Interrapidísimo, Servientrega y Envía) en una sola llamada y devuelve las tres tarifas juntas para que el usuario elija. Similar a la fórmula con la que después se **liquida** la guía. 

Esta lambda se consume por medio del API Gateway. Las lambdas internas de cada transportadora, en cambio, sí se invocan por SDK de AWS.

---

## Accionador

| Método | Ruta | Auth |
| ------ | ---- | ---- |
| `POST` | `/orquestadorCotizacion` | Sin authorizer en API Gateway. El front envía `Token` y `headersecurity` como headers para el consumo. |
| `OPTIONS` | `/orquestadorCotizacion` | Preflight CORS (responde `200`) |

**Handler:** `ApiLambdaOrquestadorCotizaciones::ApiLambdaOrquestadorCotizaciones.Lambda.Handlers.ApiGatewayFunction::HandleAsync`

---

## Request

```json
{
  "Proveedores": ["inter", "servientrega", "envia"], // Transportadoras
  "CotizacionLogigho": true,        // Siempre true
  "NombreTienda": "TIENDA A COTIZAR",
  "IdTienda": "100001",
  "Common": {
    "IdProducto": 3,
    "NumeroPiezas": 1,
    "Piezas": [{ "Peso": 3, "Largo": 10, "Ancho": 10, "Alto": 10 }],  // Medidas del paquete
    "ValorDeclarado": 100000,
    // Codigos dane de las ciudades a cotizar 
    "Ciudades": {
      "Interrapidisimo": { "Origen": "11001000", "Destino": "05001000" },
      "Servientrega":    { "Origen": "11001000", "Destino": "05001000" },
      "Envia":           { "Origen": "11001000", "Destino": "05001000" }
    },
    "FormaPago": { "Interrapidisimo": 1, "Servientrega": 1, "Envia": "6" }
  }
}
```

| Campo | Tipo | Requerido | Descripción |
| ----- | ---- | --------- | ----------- |
| `Proveedores` | `string[]` | Sí | A las transportados que quieren cotizar: `inter`, `servientrega` (o `servi`), `envia`. Solo se ejecutan los listados. |
| `CotizacionLogigho` | `bool` | Sí | `true` = tarifa propia de LogiGho. |
| `NombreTienda` | `string` | No* | Nombre de la tienda. **Fallback** si no llega `IdTienda`. |
| `IdTienda` | `string` | No* | Id de la tienda. **Se prioriza sobre el nombre**. |
| `Common.IdProducto` | `int` | Sí | Tipo de producto. |
| `Common.NumeroPiezas` | `int` | Sí | Cantidad de bultos. |
| `Common.Piezas[]` | `array` | Sí | Peso y dimensiones por pieza. El **peso total** = suma de `Peso`; define si aplica kilo adicional. |
| `Common.Piezas[].Peso` | `decimal` | Sí | Peso en kilos. |
| `Common.Piezas[].Largo/Ancho/Alto` | `decimal` | Sí | Dimensiones en cm. |
| `Common.ValorDeclarado` | `decimal` | Sí | Valor de la mercancía |
| `Common.Ciudades.<Transportadora>.Origen` | `string` | Sí | Ciudad origen en **código DANE 8** (ej. `11001000` = Bogotá). |
| `Common.Ciudades.<Transportadora>.Destino` | `string` | Sí | Ciudad destino en **código DANE 8** (ej. `05001000` = Medellín). |
| `Common.FormaPago.Interrapidisimo` | `int` | Sí | Código de forma de pago de Inter. |
| `Common.FormaPago.Servientrega` | `int` | Sí | Código de forma de pago de Servientrega. |
| `Common.FormaPago.Envia` | `string` | Sí | Código de forma de pago de Envía. |

> Para cotizar una sola transportadora, envía únicamente esa en `Proveedores` con sus `Ciudades`/`FormaPago`. Ej: `"Proveedores": ["servientrega"]`.

---

## Response

### Exitoso

```json
{
  "interrapidisimo": {
    "data": [{ "valorTotal": 27098.00, "valorEnvio": 19180.00, "fechaEntrega": "2026-07-16T18:00:00" }],
    "error": null
  },
  "servientrega": {
    "data": { "ValorFlete": 21550.00, "ValorSobreFlete": 0, "ValorTotal": 21550.00, "Informacion": "Tarifa Logigho" },
    "error": null
  },
  "envia": {
    "data": { "respuesta": "Tarifa Logigho", "valor_flete": 29950.00, "k_cobrados": 0, "dias_entrega": 0 },
    "error": null
  }
}
```

**La tarifa final a mostrar:**

| Transportadora | Ruta en el response |
| -------------- | ------------------- |
| Interrapidísimo | `interrapidisimo.data[0].valorTotal` |
| Servientrega | `servientrega.data.ValorTotal` |
| Envía | `envia.data.valor_flete` |

Cada proveedor devuelve `null` si no fue solicitado. Si una rama falla, devuelve su `error` y **las demás siguen respondiendo**.

### Errores

| Código | Cuándo |
| ------ | ------ |
| `400` | Body vacío |
| `405` | Método distinto de `POST` |
| `500` | Error interno, excepcion no controlada |

---

## Qué hace cada capa

| Capa | Archivo | Responsabilidad |
| ---- | ------- | --------------- |
| **Lambda/Handlers** | `ApiGatewayFunction.cs` | Adaptador HTTP: valida método, deserializa el body, arma la respuesta con CORS. |
| **Lambda/Handlers** | `DirectFunction.cs` | **El orquestador**: decide proveedores, precarga costos, lanza las 3 ramas en paralelo, consolida el resultado. |
| **Lambda** | `AwsLambdaInvoker.cs` | Invoca otras lambdas por SDK (`RequestResponse`) con timeout. |
| **Aplicacion/Mapeo** | `FromCommonMapper.cs` | Traduce el `Common` (formato neutro) al request específico de cada transportadora. |
| **Aplicacion/Services** | `LogighoPricingService.cs` | **La fórmula de negocio**: valida los costos de cada transportadora. |
| **Aplicacion/Interfaces** | `ILogighoPricingRepository.cs` | Puerto: el servicio depende de esta abstracción. |
| **Dominio/Entidades** | `OrchestratorEnvelope.cs` | Contrato de entrada (el body). |
| **Dominio/Comun** | `CommonShipment.cs` | El envío en formato neutro (ciudades, piezas, valor, forma de pago). |
| **Dominio/Modelos** | `InterModels`, `ServiModels`, `EnviaModels` | Request/Response de cada transportadora. |
| **Dominio/Modelos** | `MongoDbModels.cs` | `CostoTransporteTienda`, `TarifaInter`, `CiudadMongo`. |
| **Dominio/Modelos** | `SerializadorDatosDecimal.cs` | Serializer que tolera decimales que en Mongo vienen como **texto o número**. |
| **Infraestructura/Repositorio** | `LogighoPricingRepository.cs` | Consultas a MongoDB. |
| **Infraestructura/Utilidades** | `ConnectionManager.cs` | Conexión a Mongo: lazy, pooling (max 10), descanso para cold start, desencripta la cadena. |


### Resolución de la tienda (`BuscarCostoEnMemoria`)

Orden de búsqueda en `CostosTransporteTienda` (siempre con `EstadoGuia = ENTREGA`):

1. Por **`IdTienda`** + transportadora → robusto, camino preferido.
2. Por **`NombreTienda`** + transportadora → fallback si no vino el Id.
3. **Genérico**: `Generico <Transportadora> Entrega` → si la tienda no tiene configuración propia.

---

## Dependencias externas

| Servicio | Uso |
| -------- | --- |
| `MongoDB` | `CostosTransporteTienda`, `TarifasInter`, `Ciudades` |
| `Lambda: ApiLambdaCotizarInterrapidisimo` | Cotización real de Inter (`FN_INTER_COTIZAR`) |
| `Lambda: ApiLambdaGenerarCotizacion` | Cotización real de Servientrega (`FN_GENERAR_COTIZACION`) |

### Variables de entorno

| Variable | Uso |
| -------- | --- |
| `CADENA_CONEXION` | Cadena de Mongo **encriptada** (AES-256-ECB). En `DEBUG` se ignora y usa `localhost:27018`. |
| `DATABASE_NAME` | Nombre de la base de datos de Mongo a usar. |
| `FN_INTER_COTIZAR` | ARN de la lambda de Inter |
| `FN_GENERAR_COTIZACION` | ARN de la lambda de Servientrega |
| `FN_ENVIA_LIQUIDACION` | ARN de la lambda de Envía (Falta por implementar, calculo manual) |
| `INVOKE_TIMEOUT_SECONDS` | Timeout de invocación y de las ramas (default `15`) |

> El rol de ejecución necesita `lambda:InvokeFunction` sobre las lambdas de Inter y Servientrega.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|-------|-------|--------|
| 2026-07-16 | Iker Acevedo | Mejora en el cotizador para que permita un flete con valores mayores a 1kg ademas conexion con cotizador de servientrega |


---

## Observaciones

- **Deuda técnica — Envía no usa su cotizador real.** Se estima geográficamente. Pendiente por integrar ya que el servicio no estaba funcionando.
- **Tests**: `test/ApiLambdaOrquestadorCotizaciones.Tests` (xUnit + Moq + FluentAssertions, net8.0) cubre las 3 fórmulas con 8 casos. El proyecto principal excluye `test\**` del compile.
