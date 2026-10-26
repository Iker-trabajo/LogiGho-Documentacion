---
autor: Iker Acevedo
fecha_creacion: 2026-06-24
ultima_actualizacion: 2026-10-26
estado: produccion
---

## Lambda: APILambdaConsultarPedidos


**Accionador:** API Gateway

**AOT:** No

---

## ¿Qué hace?

Expone un endpoint REST para consultar pedidos paginados desde MongoDB (colección `PedidosInter`). El token Cognito lo valida el authorizer de API Gateway; la lambda usa el `sub` del token para consultar las tiendas autorizadas del usuario en la colección `ApiUsuariosTiendas` (migración desde claims JWT — ver [ADR-001](adr-001-migracion-validacion-tiendas.md)), y filtra los pedidos únicamente a las tiendas permitidas. Soporta múltiples tiendas por usuario y lógica de "acceso a todas" (`AccesoATodas: true` → comodín `*`). Los campos devueltos en cada pedido son dinámicos y se controlan desde la colección `ConfiguracionCamposApi` sin necesidad de redesplegar la lambda.

---

## Accionador

| Método | Ruta | Autenticacion |
| ------ | ---- | ------------- |
| `GET` | API Gateway — `APILambdaConsultarPedidos` | Token Cognito (IdToken) |

---

## Authorizer de API Gateway (Cognito)

La ruta `GET /pedidos` tiene un **JWT authorizer de Cognito** (`CognitoUsuariosExternos`) delante de la lambda. Ese guardia valida el token **antes** de que la función se ejecute. Si el token no es válido, API Gateway corta con `401 {"message":"Unauthorized"}` — ese mensaje viene del Gateway, no de la lambda.

| Qué espera el authorizer | Valor |
| --- | --- |
| Header | `Authorization: Bearer <token>` (no `Token`) |
| Issuer | `us-east-1_rTVHpDam9` (PreProd) |
| Audience (client_id) | `3umhmbu9l5r92g617l1467h9vf` (PreProd) |

!!! warning "Dos capas de autorización"
    1. **El authorizer de Cognito** (puerta del Gateway) lee el token de `Authorization` y valida firma/issuer/audience. La lambda nunca se ejecuta si esto falla.
    2. **La lambda** lee el token (de `Token` o `Authorization`) solo para el `sub`, y con él consulta las tiendas en `ApiUsuariosTiendas`.

    El `COGNITO_CLIENT_ID` de `APILambdaObtenerToken` coincide con el audience del authorizer: el token que emite `/token` sirve para pasar el authorizer. Para probar, el token **debe ir en `Authorization: Bearer`**, aunque la lambda también acepte `Token` — si va solo en `Token`, el authorizer corta antes con `401`.

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
| `401` | Token ausente o inválido (lo corta el authorizer de Cognito en el Gateway, antes de la lambda) |
| `403` | Usuario no existe en `ApiUsuariosTiendas`, está inactivo, o sin tiendas asignadas |
| `405` | Método HTTP distinto de GET |
| `500` | Excepción no controlada |

```json
{
  "Error": true,
  "Mensaje": "El usuario no tiene permisos asignados o esta inactivo"
}
```

---

## Flujo interno

```
FunctionHandler (Function.cs)
  -> [REQUEST] Extrae HttpMethod del request (REST API v1: input.HttpMethod)
  -> Valida método GET
  -> Extrae JWT del header Token o Authorization (Bearer)
  -> JwtClaimsHelper.ExtraerIdentidadUsuario(jwt)
       -> Decodifica JWT (sin validar firma — Cognito ya lo hizo)
       -> Extrae: sub (Cognito user ID), email, username
  -> [AUTH] UsuarioTiendaRepository.ObtenerPermisosPorSubAsync(sub)
       -> Consulta MongoDB tabla 'UsuarioTienda' por Sub
       -> Retorna: { Sub, Username, Email, AccesoATodas, Tiendas[], Activo }
       -> Valida: usuario existe, está activo
       -> Si AccesoATodas=true → idTiendas = ["*"], nombreTiendas = ["*"]
       -> Si no → extrae lista explícita de Tiendas[].IdTienda
  -> Parsea query params: page, pageSize, fechaDesde, fechaHasta, estado, numeropreenvio, transportadora, telefono
  -> ConsultarPedidosUseCase.EjecutarAsync(idTiendas, nombreTiendas, ...)
       -> MongoDB: lee ConfiguracionCamposApi → lista de campos activos (Activo: true)
       -> DocumentRepository.ObtenerPedidosPorTiendaAsync(coleccion, idTiendas, nombreTiendas, ...)
            -> Si idTiendas contiene "*" → sin filtro de tienda
            -> Si no → Filter.In("Idtienda"/"IdTienda", idTiendas)
            -> Filter.In("Tienda"/"tienda", nombreTiendas)
            -> Filtro de fechas via ObjectId hex (usa índice _id_)
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
├── Function.cs                              ← Entry point, HTTP handling, auth
├── Dominio/
│   ├── Interfaces/
│   │   ├── IDocumentRepository.cs           ← Contrato acceso pedidos
│   │   └── IUsuarioTiendaRepository.cs      ← Contrato acceso usuarios/tiendas
│   └── Modelos/
│       └── UsuarioTienda.cs                 ← Entity: Sub, Username, Email, AccesoATodas, Tiendas[]
├── Aplicacion/
│   ├── CasosDeUso/
│   │   └── ConsultarPedidosUseCase.cs       ← Lógica de negocio y paginación
│   └── DTO/
│       └── RespuestaGeneral.cs              ← Estructura de respuesta
└── Infrastructura/
    ├── Repositorio/
    │   ├── DocumentRepository.cs            ← Acceso a MongoDB colección Pedidos
    │   └── UsuarioTiendaRepository.cs       ← Acceso a MongoDB tabla UsuarioTienda
    └── Utilidades/
        └── JwtClaimsHelper.cs               ← Extracción de claims del JWT
```

---

## Multi-tienda y AccesoATodas

### Antes (JWT claims)
Viajaban en `custom:idTienda` y `custom:nombreTienda` separados por coma.

### Ahora (MongoDB UsuarioTienda)
La tabla `UsuarioTienda` tiene:
- `Tiendas[]`: lista de objetos `{ IdTienda, NombreTienda }`
- `AccesoATodas: bool`: si `true`, usuario accede a **todas** las tiendas (comodín `*`)

**Flujo:**
- Si `AccesoATodas=true` → sin filtro de tienda en query MongoDB
- Si `AccesoATodas=false` → filtro `Filter.In("Idtienda", tiendas[].IdTienda)`

El repositorio usa `Filter.In` con la lista completa — una sola tienda, cien tiendas, o todas, usan el mismo código sin ramificaciones. Ventaja vs. claims: **sin límite de tamaño en JWT**.

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
| `COLECCION_PEDIDOS` | Nombre de la colección de pedidos | `"PedidosInter"` (default) |

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
| `Amazon Cognito` | Emisión del JWT (solo se extrae el `sub`, no se usan claims personalizados) |
| `MongoDB Atlas` | Tabla `UsuarioTienda` para validar accesos; colección `Pedidos*` para datos |

---

## Historial de cambios

| Fecha | Autor | Cambio |
| ----- | ----- | ------ |
| 2026-10-26 | Iker Acevedo | **CI/CD propio del sub-dominio** en Azure DevOps (CI + CD PreProd/Prod). Ver [CI/CD y despliegue](cicd-despliegue.md). |
| 2026-10-26 | Iker Acevedo | Documentado el **authorizer de Cognito** de API Gateway (`CognitoUsuariosExternos`): el token debe ir en `Authorization: Bearer`; la lambda nunca corre si el authorizer corta. |
| 2026-10-26 | Iker Acevedo | Colección de permisos confirmada como `ApiUsuariosTiendas`; `403` cuando el usuario no existe o está inactivo. |
| 2026-10-26 | Iker Acevedo | `function-name` corregido (`ApiLambaConsultarPedidos` → `APILambdaConsultarPedidos`); secretos fuera de los `aws-lambda-tools-defaults.json` (van a variable groups de Azure Library). |
| 2026-10-26 | Iker Acevedo | `COLECCION_PEDIDOS` por defecto pasa a `PedidosInter`. Refactor: `CalcularFechaDesdeEfectiva` extraída a `PoliticaFechas` (clase de dominio pura). |
| 2026-10-26 | Iker Acevedo | Soporte de novedades como arreglo (`MotivoNovedadInter`). Ver [Novedades Inter](motivo-novedad-inter.md). |
| 2026-09-19 | Iker Acevedo | **Migración: validación de tiendas de claims JWT → MongoDB tabla `UsuarioTienda`.** Permite revocación inmediata, auditabilidad y UI de gestión. Ver [ADR-001](adr-001-migracion-validacion-tiendas.md). |
| 2026-09-19 | Iker Acevedo | Nueva interface `IUsuarioTiendaRepository` y implementación `UsuarioTiendaRepository` para consultar permisos desde BD. |
| 2026-09-19 | Iker Acevedo | Nuevo dominio `UsuarioTienda`: modelo de usuario con tiendas autorizadas e indicador `AccesoATodas`. |
| 2026-09-19 | Iker Acevedo | Cambio en `JwtClaimsHelper`: ahora extrae solo `sub`, `email`, `username`. Tiendas se obtienen de `UsuarioTiendaRepository`. |
| 2026-06-24 | Iker Acevedo | Migración a Clean Architecture: separación en Dominio / Aplicacion / Infrastructura. |
| 2026-06-24 | Iker Acevedo | Fix API Gateway: migrado de HTTP API v2 a REST API v1 corrigiendo `NullReferenceException` en `input.HttpMethod`. |
| 2026-06-24 | Iker Acevedo | Fix handler: corregida discrepancia de mayúsculas entre namespace y handler configurado en Lambda. |
| 2026-06-24 | Iker Acevedo | Campos dinámicos: proyección desde `ConfiguracionCamposApi` sin necesidad de redesplegar. |
| 2026-06-24 | Iker Acevedo | Filtro de fechas via ObjectId hex para aprovechar índice `_id_`. |
| 2026-06-24 | Iker Acevedo | Conexión a MongoDB Atlas con cadena de conexión encriptada AES-256-ECB. |

---

## Observaciones

- El token se acepta tanto en el header `Token` como en `Authorization: Bearer` para compatibilidad con distintos clientes.
- La firma del JWT no se valida en la lambda — Cognito ya la validó al emitirlo. Solo se leen los claims del payload.
- **Cambio importante (2026-09-19):** El `sub` del JWT se usa únicamente para identificar al usuario en BD. Ya **no** se extraen tiendas de claims personalizados. Ver [ADR-001](adr-001-migracion-validacion-tiendas.md).
- El filtro de tienda usa doble variante (`Idtienda` / `IdTienda` y `Tienda` / `tienda`) para tolerar inconsistencias de capitalización en la colección MongoDB.
- `pageSize` tiene un tope de 500 para proteger la memoria de la lambda (1024 MB).
- Consulta a `UsuarioTienda` tiene timeout de 5s para evitar bloqueos. Si falla, se retorna `403 Forbidden`.
- Recomendación: crear índice en MongoDB: `db.UsuarioTienda.createIndex({ "Sub": 1 })` para optimizar lookups.
