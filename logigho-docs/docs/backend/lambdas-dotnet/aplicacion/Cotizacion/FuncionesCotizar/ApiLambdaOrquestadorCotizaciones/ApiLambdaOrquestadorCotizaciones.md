## Autor: Iker Acevedo
Fecha creación: 2026-07-16
Estado: produccion
---

## Lambda: ApiLambdaOrquestadorCotizaciones
**Trigger:** API Gateway
**AOT:** No
---

## ¿Qué hace?

Orquesta las cotizaciones con cada transportadora integrada en el sistema LogiGho, actualmente estan (Interrapidísimo, Servientrega y Envía). Dentro de la misma llamada se comunica con las transportadoras mediante peticion HTTP cada transportadora tiene su propia lambda invocada por SDK, el orquestador su funcion principal es invocar a cada transportadora y asignar sus costos tal cual como se genera en la liquidacion . Incluye el concepto de la guia con o sin recaudo para hacer la cotizacion mediante esas 2 formas de envio. 

---

## Accionador

| Método | Ruta | Auth |
| ------ | ---- | ---- |
| `POST` | `/orquestadorCotizacion` | Autorizacion en API Gateway. El front envía `Token` y `headersecurity` como headers para el .consumo. |
| `OPTIONS` | `/orquestadorCotizacion` | Preflight CORS (responde `200`) |

**Handler:** 

`ApiLambdaOrquestadorCotizaciones::ApiLambdaOrquestadorCotizaciones.Lambda.Handlers.ApiGatewayFunction::HandleAsync`

---

## Request

```json
{
  "Proveedores": ["inter", "servientrega", "envia"], // Transportadoras que se requieren cotizar
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
    "FormaPago": { "Interrapidisimo": 1, "Servientrega": 1, "Envia": "6" },
    "ConRecaudo": true    // Defina la forma del envio
  }
}
```

| Campo | Tipo | Requerido | Descripción |
| ----- | ---- | --------- | ----------- |
| `Proveedores` | `string[]` | Sí | A las transportados que quieren cotizar: `inter`, `servientrega` (o `servi`), `envia`. Solo se ejecutan los listados. |
| `CotizacionLogigho` | `bool` | Sí | `true` = tarifa propia de LogiGho. |
| `NombreTienda` | `string` | Sí* | Nombre de la tienda. **Fallback** si no llega `IdTienda`. |
| `IdTienda` | `string` | Sí* | Id de la tienda. **Se prioriza sobre el nombre**. |
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
| `Common.ConRecaudo` | `bool` | Sí | El switch global de la cotización. Afecta a las **3** transportadoras a la vez (ver [Con/sin recaudo](#consin-recaudo) abajo). |

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
| **Lambda/Handlers** | `DirectFunction.cs` | **El orquestador**: arma el contexto y corre el catálogo de `ICotizadorProveedor` en paralelo. Ver [Patrón Strategy](#patron-strategy-icotizadorproveedor). |
| **Lambda** | `AwsLambdaInvoker.cs` | Invoca otras lambdas por SDK (`RequestResponse`) con timeout. |
| **Aplicacion/Mapeo** | `FromCommonMapper.cs` | Traduce el `Common` (formato neutro) al request específico de cada transportadora. |
| **Aplicacion/Cotizadores** | `CotizadorEnvia.cs`, `CotizadorInter.cs`, `CotizadorServientrega.cs` | Implementaciones de `ICotizadorProveedor`, una por transportadora. |
| **Aplicacion/Cotizadores** | `RegistroCotizadores.cs` | Catálogo de proveedores disponibles. |
| **Aplicacion/Services** | `LogighoPricingService.cs` | **La fórmula de negocio**: calcula la tarifa Logigho con/sin recaudo sobre el flete real de cada transportadora. |
| **Aplicacion/Services** | `EnviaCuentaResolver.cs` | Resuelve qué cuenta de Envía usar según `ConRecaudo`. Ver [Cuentas de Envía](#cuentas-de-envia-enviacuentaresolver). |
| **Aplicacion/Interfaces** | `ILogighoPricingRepository.cs` | Puerto: el servicio depende de esta abstracción. |
| **Dominio/Entidades** | `OrchestratorEnvelope.cs` | Contrato de entrada (el body). |
| **Dominio/Comun** | `CommonShipment.cs` | El envío en formato neutro (ciudades, piezas, valor, forma de pago, `ConRecaudo`). |
| **Dominio/Modelos** | `InterModels`, `ServiModels`, `EnviaModels` | Request/Response de cada transportadora. |
| **Dominio/Modelos** | `MongoDbModels.cs` | `CostoTransporteTienda`, `TarifaInter`, `CiudadMongo`. |
| **Dominio/Modelos** | `SerializadorDatosDecimal.cs` | Serializer que tolera decimales que en Mongo vienen como **texto o número**. |
| **Infraestructura/Repositorio** | `LogighoPricingRepository.cs` | Consultas a MongoDB. |
| **Infraestructura/Utilidades** | `ConnectionManager.cs` | Conexión a Mongo: lazy, pooling (max 10), descanso para cold start, desencripta la cadena. |

---

## Patrón Strategy (`ICotizadorProveedor`)

Antes, `DirectFunction` tenía un bloque `if/else` duplicado por transportadora (~200 líneas). Se refactorizó a **Strategy**: cada transportadora es una clase que implementa `ICotizadorProveedor` (mapea su request, invoca su lambda y aplica la fórmula Logigho si corresponde), y `RegistroCotizadores` es el catálogo. `DirectFunction` quedó en ~55 líneas: arma el contexto común y corre el catálogo en paralelo.

**Agregar una transportadora nueva = 1 clase nueva + 1 línea en `RegistroCotizadores`.** No se toca el orquestador.

Cada `CotizadorXxx` tiene un constructor `internal` alterno que recibe el invoker mockeado, para poder testear sin llamar a AWS de verdad.

> 💡 **Patrón Strategy**: define una familia de algoritmos intercambiables detrás de una interfaz común, para que el código que los usa no necesite saber cuál es cuál. Es como enchufes de pared distintos para el mismo tomacorriente: el aparato (el orquestador) no le importa qué transportadora hay detrás, solo que cumple el contrato.

---

## Con/sin recaudo

`Common.ConRecaudo` es el switch que llega desde el front en cada cotización y afecta a las 3 transportadoras:

| Transportadora | Con recaudo | Sin recaudo |
| --------------- | ----------- | ----------- |
| Interrapidísimo | `AplicaContrapago = true` | `AplicaContrapago = false` |
| Servientrega | `EnvioConCobro = true` | `EnvioConCobro = false` |
| Envía | Cuenta `ENVIA_CUENTA_RECAUDO` (`cod_formapago = 4`, Crédito) | Cuenta `ENVIA_CUENTA_SIN_RECAUDO` (`cod_formapago = 7`, Contraentrega) |

⚠️ **Envía tiene la forma de pago invertida respecto al resto del mercado**: en Envía, Crédito = CON recaudo y Contraentrega = SIN recaudo. Ver `EnviaCuentaResolver` abajo y la tabla completa de forma de pago en el ADR de este módulo.

### Calculo cotizacion

- `fleteBase`
- **Con recaudo:** `%Recaudo + %Seguro`.
- **Sin recaudo:** solo `%Seguro` 


### Cuentas de Envía (`EnviaCuentaResolver`)

Envía no maneja "con/sin recaudo" como un flag suelto: son **2 cuentas distintas configuradas del lado de Envía**.

---

### Costos de transporte de la tienda (`BuscarCostoEnMemoria`)

Orden de búsqueda en `CostosTransporteTienda` (siempre con `EstadoGuia = ENTREGA`):

1. Por **`IdTienda`** + transportadora → robusto, camino preferido.
2. Por **`NombreTienda`** + transportadora → fallback si no vino el Id.
3. **Genérico**: `Generico <Transportadora> Entrega` → si la tienda no tiene configuración propia.

---

## Resumen: cómo se autentica cada lambda hija contra su transportadora

Las 3 lambdas (`ApiLambdaCotizarInterrapidisimo`, `ApiLambdaGenerarCotizacion`, `ApiLambdaCotizarEnvia`) las invoca este orquestador **por SDK de AWS** (`lambda:InvokeFunction`, sin body HTTP — el rol de ejecución IAM es la autenticación entre lambdas). Puertas afuera, cada una habla con su transportadora de forma distinta:

| Transportadora | Método/URL real | Autenticación contra la transportadora |
| --------------- | ---------------- | ---------------------------------------- |
| Interrapidísimo | `GET` — parámetros **en el path** de la URL | 2 headers fijos por request: `x-app-signature` + `x-app-security_token`. No hay login ni token que expire. |
| Servientrega | `POST` (login) + `POST` (cotización) | Login con usuario/contraseña (Sisclinet) devuelve un **Bearer token** que se usa en la cotización. La lambda se autogestiona el token si no le llega uno. |
| Envía | `POST` — body JSON | **Sin token ni API key** — la identidad de la cuenta va en el body (`cod_regional_cta`/`cod_oficina_cta`/`cod_cuenta`), y esos 3 datos cambian según con/sin recaudo. |

Ver la sección **"Cómo se consume la API real de..."** en la doc de cada lambda para el detalle completo (URL exacta, headers, ejemplo para probar a mano): [Interrapidísimo](../ApiLambdaCotizarInterrapidisimo/ApiLambdaCotizarInterrapidisimo.md#cómo-se-consume-la-api-real-de-interrapidísimo), [Servientrega](../ApiLambdaGenerarCotizacion/ApiLambdaGenerarCotizacion.md#cómo-se-consume-la-api-real-de-servientrega), [Envía](../ApiLambdaCotizarEnvia/ApiLambdaCotizarEnvia.md#cómo-se-consume-la-api-real-de-envía).

---

## Dependencias externas

| Servicio | Uso |
| -------- | --- |
| `MongoDB` | `CostosTransporteTienda`, `TarifasInter`, `Ciudades` |
| `Lambda: ApiLambdaCotizarInterrapidisimo` | Cotización real de Inter (`FN_INTER_COTIZAR`) |
| `Lambda: ApiLambdaGenerarCotizacion` | Cotización real de Servientrega (`FN_GENERAR_COTIZACION`) |
| `Lambda: ApiLambdaCotizarEnvia` | Cotización real de Envía (`FN_ENVIA_LIQUIDACION`) |

### Variables de entorno

| Variable | Uso |
| -------- | --- |
| `CADENA_CONEXION` | Cadena de Mongo **encriptada** (AES-256-ECB). En `DEBUG` se ignora y usa `localhost:27018`. |
| `DATABASE_NAME` | Nombre de la base de datos de Mongo a usar. |
| `FN_INTER_COTIZAR` | ARN de la lambda de Inter |
| `FN_GENERAR_COTIZACION` | ARN de la lambda de Servientrega |
| `FN_ENVIA_LIQUIDACION` | ARN de la lambda de Envía |
| `ENVIA_CUENTA_RECAUDO` | Cuenta de Envía a usar cuando `ConRecaudo = true` |
| `ENVIA_CUENTA_SIN_RECAUDO` | Cuenta de Envía a usar cuando `ConRecaudo = false` |
| `INVOKE_TIMEOUT_SECONDS` | Timeout de invocación y de las ramas (default `15`) |

> El rol de ejecución necesita `lambda:InvokeFunction` sobre las lambdas de Inter, Servientrega y Envía.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|-------|-------|--------|
| 2026-09-09 | Iker Acevedo | **Envía pasa a cotizar real** (se deja de estimar geográfico) y las 3 transportadoras soportan `ConRecaudo`. Refactor a patrón Strategy (`ICotizadorProveedor` + `RegistroCotizadores`), `DirectFunction` reducido de ~200 a ~55 líneas. Nuevo `EnviaCuentaResolver` para las 2 cuentas de Envía. 45 tests xUnit nuevos. |
| 2026-07-16 | Iker Acevedo | Mejora en el cotizador para que permita un flete con valores mayores a 1kg ademas conexion con cotizador de servientrega |

---

## Observaciones

- **Pendiente / fuera de esta feature**:Aun no se implementa la guia sin recaudo en la creacion de pedidos de envia, toca siempre generarla con un valor minimo de recaudo. 
- `Trayecto` de Envía: Dentro de la respuesta de la cotización de envia 
- **Validación en preprod**: Se hicieron pruebas end to end validando cada transportadora y avalado por el area financiero.
- **Tests**: `test/ApiLambdaOrquestadorCotizaciones.Tests` (xUnit + Moq + FluentAssertions, net8.0). El proyecto principal excluye `test\**` del compile.
- Ver [ADR — Strategy para cotizadores y semántica de recaudo](ADR-001-strategy-cotizadores.md) para el porqué del refactor y la tabla completa de forma de pago por transportadora.
