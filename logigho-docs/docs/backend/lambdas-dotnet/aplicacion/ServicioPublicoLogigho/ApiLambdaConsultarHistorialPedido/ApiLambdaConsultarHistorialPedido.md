---
autor: Iker Acevedo
fecha_creacion: 2026-10-27
ultima_actualizacion: 2026-10-27
estado: probado-preprod
---

## Lambda: ApiLambdaConsultarHistorialPedido

**Accionador:** API Gateway

**AOT:** No · **Runtime:** `dotnet10`

---

## ¿Qué hace?

Expone `GET /pedidos/{numeropreenvio}/historial` para que un integrador externo consulte, **en vivo**, el historial completo de estados de una guía de Interrapidísimo. Devuelve 6 campos por evento: `nombreEstado`, `idLocalidadOrigen`, `idLocalidadDestino`, `idLocalidadEnCurso`, `fechaEstado`, `descripcionAsociada`.

Interrapidísimo es un tercero con credencial compartida, así que la lambda lo protege con cuatro capas: límite por usuario (minuto y día), presupuesto global hacia Inter, caché corta (90 s) y circuit breaker. Al exceder un límite responde **429**. A Inter solo se llega si la guía pertenece a una tienda del usuario; una guía ajena o inexistente devuelve el **mismo 404** (no revela si existe).

---

## Accionador

| Método | Ruta | Autenticación |
| ------ | ---- | ------------- |
| `GET` | `/pedidos/{numeropreenvio}/historial` | Token Cognito (IdToken) |

Mismo authorizer de Cognito (`CognitoUsuariosExternos`) que [ConsultarPedidos](../ApiLambdaConsultarPedidos/ApiLambdaConsultarPedidos.md): el token va en `Authorization: Bearer`; si falta, el Gateway corta con 401 antes de la lambda.

---

## Request

Sin query params ni body. La tienda sale del pedido, nunca del cliente.

| Parámetro | Dónde | Regla |
| --------- | ----- | ----- |
| `numeropreenvio` | path | solo dígitos, 8-15 |

---

## Response

### Exitoso (200)

```json
{
  "Error": false,
  "Mensaje": null,
  "Resultado": {
    "numeroPreenvio": 240064173875,
    "estados": [
      {
        "nombreEstado": "Telemercadeo",
        "idLocalidadOrigen": "05001000",
        "idLocalidadDestino": "41524000",
        "idLocalidadEnCurso": "41001000",
        "fechaEstado": "2026-10-26T18:45:13.563",
        "descripcionAsociada": ""
      }
    ]
  }
}
```

`estados` viene **más reciente primero** (orden nativo de Inter). Header `X-Cache: HIT|MISS`.

### Errores

| Código | Cuándo | Headers extra |
| ------ | ------ | ------------- |
| `400` | formato de guía inválido | |
| `401` | sin token / `sub` vacío (el authorizer corta antes para token ausente) | |
| `403` | usuario inactivo o sin tiendas | |
| `404` | guía ajena o inexistente (idéntico, no revela cuál) | |
| `405` | método != GET | |
| `429` | excede límite por usuario o tope global hacia Inter | `Retry-After`, `X-RateLimit-Limit`, `X-RateLimit-Remaining` |
| `502` | Inter falló | |
| `503` | breaker abierto (Inter inestable) | |

---

## Flujo interno (cada paso corta; a Inter solo se llega en el paso 8)

```
FunctionHandler
  1. método != GET -> 405
  2. (authorizer Cognito del gateway: token ausente -> 401 antes de la lambda)
  3. sub del JWT (JwtClaimsHelper.ExtraerIdentidadUsuario) vacío -> 401
  4. rate limit por usuario (minuto/día, LimiteConsultasHistorial) excedido -> 429 + Retry-After
  5. ValidadorGuia (dígitos, 8-15) inválido -> 400 (no toca DB)
  6. permisos en ApiUsuariosTiendas por sub; nulo/inactivo/sin tiendas -> 403
  7. propiedad: PedidosInter con esa guía + tienda + INTERRAPIDISIMO; ajena/inexistente -> 404 uniforme
  8. caché CacheHistorialInter; hit -> 200 X-Cache: HIT
  9. presupuesto global Inter + breaker -> 429 / 503
 10. POST Inter ClientesCredito/ConsultarEstadosGuiasCliente (1 guía, timeout 8s, 1 reintento)
 11. mapear 6 campos + guardar en caché -> 200
```

---

## Arquitectura (Clean Architecture)

```
ApiLambdaConsultarHistorialPedido/
├── Function.cs                        ← handler, GET-only, path param, mapea status->headers
├── Dominio/                           ← lógica pura, testeable sin Mongo ni HTTP
│   ├── ValidadorGuia.cs
│   ├── PoliticaRateLimit.cs           ← permitido/429 + Retry-After (minuto/día/global)
│   ├── PoliticaPropiedad.cs           ← 200/403/404 uniforme
│   ├── MapeadorHistorial.cs           ← respuesta Inter -> EstadoDto (idLocalidadenCurso)
│   ├── PoliticaBreaker.cs / PoliticaCache.cs
│   ├── Modelos/UsuarioTienda.cs
│   └── Interfaces/ (5 contratos)
├── Aplicacion/
│   ├── CasosDeUso/ConsultarHistorialUseCase.cs   ← orquesta pasos 3-11
│   ├── OpcionesHistorial.cs           ← límites desde variables de entorno
│   └── DTO/ (RespuestaGeneral, HistorialDto, EstadoDto)
└── Infrastructura/
    ├── Repositorio/ (ConexionMongo, UsuarioTienda, Pedido, RateLimit, Cache)
    ├── Servicios/ClienteInterHistorial.cs        ← HttpClient estático, timeout 8s, 1 reintento
    └── Utilidades/JwtClaimsHelper.cs
```

---

## Protección de Interrapidísimo (4 capas)

| Capa | Mecanismo | Variable |
| --- | --- | --- |
| A — por usuario | contador ventana fija en Mongo (minuto y día), `$inc` atómico | `LIMITE_CONSULTAS_POR_MINUTO` (20), `LIMITE_CONSULTAS_POR_DIA` (1000) |
| B — global + breaker | contador global/min (solo llamadas reales) + breaker en Mongo | `LIMITE_GLOBAL_INTER_POR_MINUTO` (60), `BREAKER_FALLAS` (5), `BREAKER_SEGUNDOS` (30) |
| C — caché | `CacheHistorialInter`, TTL corto | `TTL_CACHE_HISTORIAL_SEG` (90) |
| D — infra (la pone Iker) | concurrencia reservada + throttling del gateway | — |

El timeout a Inter es corto (`TIMEOUT_INTER_SEG`, 8 s) con 1 reintento, porque el API Gateway corta a los **29 s**.

---

## Variables de entorno (solo nombres; valores en variable group)

`CADENA_CONEXION`, `DATABASE_NAME`, `COLECCION_PEDIDOS`, `URL_SERVICIO_GUIA`, `ID_CLIENTE`, `USER_AUTH`, `TOKEN_AUTH`, `TTL_CACHE_HISTORIAL_SEG`, `LIMITE_CONSULTAS_POR_MINUTO`, `LIMITE_CONSULTAS_POR_DIA`, `LIMITE_GLOBAL_INTER_POR_MINUTO`, `TIMEOUT_INTER_SEG`, `BREAKER_FALLAS`, `BREAKER_SEGUNDOS`.

---

## Colecciones Mongo nuevas

- `CacheHistorialInter`: `{ _id: guía, Json, ExpiraEn }`, índice TTL sobre `ExpiraEn`.
- `LimiteConsultasHistorial`: contadores (`{sub}:{min}`, `{sub}:{día}`, `inter:{min}`) + `breaker:inter`, índice TTL sobre `ExpiraEn`.

---

## Cómo autenticarse (integradores)

1. Pide un token con `POST /token` (ver [ConsultarPedidos](../ApiLambdaConsultarPedidos/ApiLambdaConsultarPedidos.md)) usando tus credenciales de servicio.
2. Copia el `IdToken` que devuelve.
3. En la llamada al historial manda el header `Authorization: Bearer <IdToken>` (con la palabra `Bearer` y un espacio antes del token).

El token de Cognito dura ~1 hora. Un **401 Unauthorized** significa token ausente, vencido o inválido (lo corta el authorizer, no llega a la lambda). Un **403** significa que tu usuario no tiene tiendas asignadas o está inactivo. Un **404** es una guía que no pertenece a tus tiendas (o no existe): el mensaje es el mismo a propósito, para no revelar qué guías existen.

> **Nota de integración:** tu usuario de servicio debe existir en la plataforma con tus tiendas asignadas (o acceso global). El `sub` de tu token debe coincidir con el del usuario registrado; si usas un token de otro usuario, verás 403.

---

## Historial de cambios

| Fecha | Autor | Cambio |
| ----- | ----- | ------ |
| 2026-10-27 | Iker Acevedo | Creación de la lambda (historia 015, spec 003). Consulta en vivo a Inter con límites + caché + 404 por propiedad. net10. 46 tests verdes. |
| 2026-10-27 | Iker Acevedo | **Probado en PreProd:** `GET /pedidos/{guia}/historial` → 200 OK con estados mapeados. Infra desplegada como código (Terraform: función, rol, log group, concurrencia). Ruta del gateway creada a mano (deuda M-118). Pendiente: salida a producción (crear la ruta en el REST v1 `ApiIntegracionesExternas`). |
