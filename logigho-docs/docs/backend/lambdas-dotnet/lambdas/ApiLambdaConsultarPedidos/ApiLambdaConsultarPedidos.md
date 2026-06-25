## Autor: Iker Acevedo
Fecha creacion: 2026-06-24

Estado: produccion

## Lambda: APILambdaConsultarPedidos


**Accionador:** API Gateway

**AOT:** No

---

## ¿Qué hace?

Expone un endpoint REST para consultar pedidos paginados desde MongoDB (colección `PedidosInter`). Valida el token Cognito del request, extrae las tiendas autorizadas que viajan en los claims de JWT y filtra los pedidos únicamente a las tiendas del usuario. Soporta múltiples tiendas por usuario (el JWT puede tener varios IDs separados por coma). Los campos devueltos en cada pedido son dinámicos y se controlan desde la colección `ConfiguracionCamposApi` en MongoDB sin necesidad de redesplegar la lambda.

---

## Accionador

| Método | Ruta | Autenticacion |
| ------ | ---- | ------------- |
| `GET` | API Gateway — `APILambdaConsultarPedidos` | Token Cognito (IdToken) |

---

## Request

### Headers

| Header | Requerido | Descripción |
| ------ | --------- | ----------- |
| `Token` | Sí (o `Authorization`) | IdToken de Cognito obtenido desde `APILambdaObtenerToken` |
| `Authorization` | Sí (o `Token`) | Alternativa: `Bearer eyJraWQi...` |

### Query Parameters

| Parámetro | Tipo | Requerido | Descripción |
| --------- | ---- | --------- | ----------- |
| `page` | `int` | No | Página a consultar. Default: `1` |
| `pageSize` | `int` | No | Registros por página. Default: `100`, máximo: `500` |
| `fechaDesde` | `string` | No | Fecha inicio en formato `yyyy-MM-dd` |
| `fechaHasta` | `string` | No | Fecha fin en formato `yyyy-MM-dd` |
| `estado` | `string` | No | Filtra por estado del pedido (ej: `"Cargado"`, `"Rechazado"`) |
| `numeropreenvio` | `string` | No | Filtra por número de preenvío exacto |
| `transportadora` | `string` | No | Filtra por nombre de transportadora |
| `telefono` | `string` | No | Filtra por teléfono del destinatario |

### Ejemplo de request

```
GET /pedidos?page=1&pageSize=50&fechaDesde=2026-01-01&fechaHasta=2026-06-24
Token: eyJraWQiOiJ...
```

---

## Response

### Exitoso (200)

```json
{
  "TotalRegistros": 208088,
  "NumeroPaginas": 4162,
  "Resultados": [
    {
      "FechaCarga": "2026-06-01",
      "IdCarga": "10000",
      "Estado": "Cargado",
      "NumeroPreenvio": "892341234",
      "Transportadora": "INTERRAPIDISIMO",
      "Nombre": "Juan Pérez",
      "Telefono": "3001234567"
    }
  ],
  "Error": false,
  "Mensaje": null
}
```

Los campos dentro de `Resultados` son dinámicos — dependen de la configuración activa en la colección `ConfiguracionCamposApi`. Solo se retornan los campos con `Activo: true`.

### Errores

| Código | Cuándo |
| ------ | ------ |
| `401` | Token ausente, inválido o sin tiendas en los claims |
| `405` | Método HTTP distinto de GET |
| `500` | Excepción no controlada |

```json
{
  "Error": true,
  "Mensaje": "No autorizado, el token no contiene tiendas validas, por favor revisar"
}
```

---

## Flujo interno

```
FunctionHandler (Function.cs)
  -> [REQUEST] Extrae HttpMethod del request (REST API v1: input.HttpMethod)
  -> Valida método GET
  -> Extrae JWT del header Token o Authorization (Bearer)
  -> JwtClaimsHelper.ExtraerClaimsUsuario(jwt)
       -> Decodifica claims del IdToken (sin validar firma — ya validada por Cognito)
       -> Extrae custom:idTienda  → split por coma → List<string> idTiendas
       -> Extrae custom:nombreTienda → split por coma → List<string> nombreTiendas
  -> [AUTH] Valida que idTiendas y nombreTiendas no estén vacíos
  -> Parsea query params: page, pageSize, fechaDesde, fechaHasta, estado, numeropreenvio, transportadora, telefono
  -> ConsultarPedidosUseCase.EjecutarAsync(idTiendas, nombreTiendas, ...)
       -> MongoDB: lee ConfiguracionCamposApi → lista de campos activos (Activo: true)
       -> DocumentRepository.ObtenerPedidosPorTiendaAsync(coleccion, idTiendas, nombreTiendas, ...)
            -> Filter.In("Idtienda"/"IdTienda", idTiendas)
            -> Filter.In("Tienda"/"tienda", nombreTiendas)
            -> Filtro de fechas via ObjectId hex (usa índice _id_ sin índice adicional)
            -> Filtros opcionales: estado, numeropreenvio, transportadora, telefono
            -> CountDocumentsAsync (total) + Find con Skip/Limit (página)
            -> Proyección dinámica de campos activos
            -> BsonTypeMapper.MapToDotNetValue → objetos .NET limpios
  -> [RESULT] Retorna TotalRegistros, NumeroPaginas, Resultados
```

---

## Arquitectura Clean Architecture

```
APILambdaConsultarPedidos/
├── Function.cs                          ← Entry point, HTTP handling, auth
├── Dominio/
│   └── Interfaces/
│       └── IDocumentRepository.cs       ← Contrato del repositorio
├── Aplicacion/
│   ├── CasosDeUso/
│   │   └── ConsultarPedidosUseCase.cs   ← Lógica de negocio y paginación
│   └── DTO/
│       └── RespuestaGeneral.cs          ← Estructura de respuesta
└── Infrastructura/
    ├── Repositorio/
    │   └── DocumentRepository.cs        ← Acceso a MongoDB
    └── Utilidades/
        └── JwtClaimsHelper.cs           ← Extracción de claims del JWT
```

---

## Multi-tienda

El JWT de Cognito puede tener una o varias tiendas separadas por coma:

```
custom:idTienda    = "156938,204710"
custom:nombreTienda = "Tienda Norte,Tienda Sur"
```

El repositorio usa `Filter.In` con la lista completa — una sola tienda o cien tiendas usan el mismo código sin ramificaciones. Esto significa que un usuario con acceso a varias tiendas obtiene los pedidos de todas en una sola consulta.

---

## Campos dinámicos (ConfiguracionCamposApi)

Los campos retornados en cada pedido se configuran en MongoDB sin redesplegar:

**Colección:** `ConfiguracionCamposApi`

```json
{ "Campo": "FechaCarga", "Activo": true }
{ "Campo": "Estado",     "Activo": true }
{ "Campo": "CampoInterno", "Activo": false }
```

Solo los documentos con `Activo: true` se incluyen en la proyección y en la respuesta. Para agregar o quitar un campo basta con cambiar el valor en MongoDB.

---

## Variables de entorno

| Variable | Descripción | Valor ejemplo |
| -------- | ----------- | ------------- |
| `CADENA_CONEXION` | Cadena de conexión MongoDB (encriptada AES-256-ECB) | String encriptado |
| `DATABASE_NAME` | Nombre de la base de datos MongoDB | `"LogighoDB"` |
| `COLECCION_PEDIDOS` | Nombre de la colección de pedidos | `"PedidosInter"` |

---

## Configuración Lambda

| Parámetro | Valor |
| --------- | ----- |
| Runtime | `dotnet8` |
| Handler | `APILambdaConsultarPedidos::ApiLambdaConsultarPedidos.Function::FunctionHandler` |
| Memory | `1024 MB` |
| Timeout | `30 segundos` |
| Architecture | `x86_64` |

---

## Dependencias externas

| Servicio | Uso |
| -------- | --- |
| `Amazon Cognito` | Emisión del JWT que esta lambda valida leyendo sus claims |

---

## Historial de cambios

| Fecha | Autor | Cambio |
| ----- | ----- | ------ |
| 2026-06-24 | Iker Acevedo | Migración a Clean Architecture: separación en Dominio / Aplicacion / Infrastructura. |
| 2026-06-24 | Iker Acevedo | Fix API Gateway: migrado de HTTP API v2 (`APIGatewayHttpApiV2ProxyRequest`) a REST API v1 (`APIGatewayProxyRequest`) corrigiendo `NullReferenceException` en `input.HttpMethod`. |
| 2026-06-24 | Iker Acevedo | Fix handler: corregida discrepancia de mayúsculas entre namespace `ApiLambdaConsultarPedidos` y handler configurado en Lambda. |
| 2026-06-24 | Iker Acevedo | Soporte multi-tienda: `custom:idTienda` y `custom:nombreTienda` del JWT se parsean como listas separadas por coma. El filtro MongoDB usa `Filter.In` para cubrir una o varias tiendas con el mismo código. |
| 2026-06-24 | Iker Acevedo | Campos dinámicos: proyección de campos leída desde `ConfiguracionCamposApi` en cada request — sin redesplegar para agregar o quitar campos. |
| 2026-06-24 | Iker Acevedo | Filtro de fechas via ObjectId hex: rango de fechas se convierte a ObjectId para aprovechar el índice `_id_` existente sin crear índice adicional. |
| 2026-06-24 | Iker Acevedo | Conexión a MongoDB Atlas: cadena de conexión encriptada AES-256-ECB en variable de entorno `CADENA_CONEXION`. |

---

## Observaciones

- El token se acepta tanto en el header `Token` como en `Authorization: Bearer` para compatibilidad con distintos clientes.
- La firma del JWT no se valida en la lambda — Cognito ya la validó al emitirlo. Solo se leen los claims del payload.
- El filtro de tienda usa doble variante (`Idtienda` / `IdTienda` y `Tienda` / `tienda`) para tolerar inconsistencias de capitalización en la colección MongoDB.
- `pageSize` tiene un tope de 500 para proteger la memoria de la lambda (1024 MB).
