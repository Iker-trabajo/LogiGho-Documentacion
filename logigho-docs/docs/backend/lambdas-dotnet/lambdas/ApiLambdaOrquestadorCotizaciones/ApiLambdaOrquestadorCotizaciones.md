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

Desde `feature/integracion-cotizador-envia` **Envía cotiza real** (antes se estimaba geográfico, ver `ApiLambdaCotizarEnvia`) y **las 3 transportadoras soportan cotización con/sin recaudo** vía `Common.ConRecaudo` (antes solo existía el concepto para Envía, y ni siquiera se aplicaba porque no se invocaba de verdad).

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
    "FormaPago": { "Interrapidisimo": 1, "Servientrega": 1, "Envia": "6" },
    "ConRecaudo": true
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

⚠️ **Envía tiene la semántica invertida respecto al resto del mercado**: en Envía, Crédito = CON recaudo y Contraentrega = SIN recaudo. Ver `EnviaCuentaResolver` abajo y la tabla completa de forma de pago en el ADR de este módulo.

### Fórmula Logigho (`LogighoPricingService`)

`fleteBase` (el mayor entre el flete real que devuelve la transportadora y la config geográfica en Mongo) `+ valorDeclarado * porcentaje`:

- **Con recaudo:** `%Recaudo + %Seguro`.
- **Sin recaudo:** solo `%Seguro` — no se cobra comisión de recaudo sobre algo que no se está recaudando.

Mismo patrón para las 3 transportadoras (`CalcularTarifaEnviaConFleteRealAsync` y sus equivalentes de Inter/Servientrega).

### Cuentas de Envía (`EnviaCuentaResolver`)

Envía no maneja "con/sin recaudo" como un flag suelto: son **2 cuentas distintas configuradas del lado de Envía**, globales para toda la plataforma (no por tienda). `EnviaCuentaResolver` elige la cuenta (`cod_regional_cta`, `cod_oficina_cta`, `cod_cuenta`) según `ConRecaudo`, leyendo `ENVIA_CUENTA_RECAUDO` / `ENVIA_CUENTA_SIN_RECAUDO`. Regional y oficina son constantes fijas en código (iguales en ambas cuentas).

---

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

- **Pendiente / fuera de esta feature**: guía con recaudo para Envía en modo sin-recaudo aún no se factura bien operativamente (se puede crear el pedido, pero falla la facturación). No es un bug de esta entrega, es un caso no soportado todavía.
- `cubrimiento` de Envía: el backend lo sigue devolviendo, pero el front decidió no mostrarlo en la UI.
- **Validación en preprod**: Envía se validó con matemática exacta contra datos reales de tienda. Inter se probó aislado. Servientrega solo con tests unitarios (mock del repositorio) — revisar si ya se validó en preprod real antes de dar esto por "100% probado en producción".
- **Tests**: `test/ApiLambdaOrquestadorCotizaciones.Tests` (xUnit + Moq + FluentAssertions, net8.0). El proyecto principal excluye `test\**` del compile.
- Ver [ADR — Strategy para cotizadores y semántica de recaudo](ADR-001-strategy-cotizadores.md) para el porqué del refactor y la tabla completa de forma de pago por transportadora.
