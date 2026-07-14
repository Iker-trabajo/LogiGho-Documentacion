## Autor: Iker Acevedo
Fecha creacion: 2026-07-13

Estado: produccion

## Lambda: ApiLambdaListarPaginasPancake

**Accionador:** Step Functions (dentro del **1er `Map`** — una iteración por cuenta madre)

**AOT:** No

**Posición en el pipeline:** 3 de 5

---

## ¿Qué hace?

Para **una cuenta madre**, consulta la API de Pancake y trae **todas sus páginas** (activas e inactivas), luego las sincroniza en MongoDB (`PancakePaginas`) con un **UPSERT** por `pageId`. Así la colección de páginas queda siempre al día antes de pedir estadísticas.

Llamada a Pancake:

```
GET https://pages.fm/api/v1/pages?access_token={token_de_la_cuenta}
```

---

## Request

Cada iteración del `Map` recibe **un elemento** del array `$.cuentas` (una cuenta madre):

```json
{ "cuenta_id": "10001", "token_acceso": "eyJhbGciOiJ..." }
```

| Campo | Tipo | Requerido | Descripción |
| ----- | ---- | --------- | ----------- |
| `cuenta_id` | `string` | Sí | Id de la cuenta madre (se guarda en cada página) |
| `token_acceso` | `string` | Sí | Token con el que se consulta `pages.fm` |

---

## Respuesta de Pancake (estructura relevante)

```json
{
  "success": true,
  "categorized": {
    "activated":   [ { "id": "850833678102371", "name": "Mi Página", "settings": { "page_access_token": "eyJ..." }, ... } ],
    "inactivated": [ ... ],
    "activated_page_ids": [ ... ]
  }
}
```

!!! warning "El token de la página está anidado"
    El `page_access_token` de cada página **no** está al nivel de la página, sino dentro de `settings.page_access_token`. Es fácil de pasar por alto.

---

## Response (de la lambda)

Devuelve un resumen de la sincronización (el pipeline no usa esta salida, solo confirma que corrió):

```json
{ "cuenta_id": "10001", "paginas_procesadas": 129, "status": "ok" }
```

El **efecto real** de esta lambda es la escritura en MongoDB, no su valor de retorno.

---

## Flujo interno

```
FunctionHandler (Function.cs)  [Map: una cuenta madre]
  -> ListarPaginasUseCase.EjecutarAsync(cuenta)
       -> IPancakeClient.ObtenerPaginasAsync(token)   [Infraestructura/Servicios]
            -> HttpClient estático GET pages.fm/api/v1/pages?access_token=...
            -> (token en la URL → NUNCA se loguea la URL completa)
       -> Mapea activated + inactivated a entidades Pagina
            -> pageId, cuentaMadreId, nombre, activada, conectada, pais,
               tokenAccesoPagina (settings.page_access_token), zonaHoraria, usuarios[]
       -> IPaginasRepository.GuardarAsync(paginas)
            -> MongoDB: PancakePaginas — BulkWrite UPSERT por pageId
  -> [RESULT] resumen de la sincronización
```

---

## Arquitectura Clean Architecture

```
ApiLambdaListarPaginasPancake/
├── Function.cs
├── Dominio/
│   └── Entidades/
│       └── Pagina.cs
├── Aplicacion/
│   ├── CasosUso/
│   │   └── ListarPaginasUseCase.cs
│   └── Interfaces/
│       ├── Repositorios/ IPaginasRepository.cs
│       └── Servicios/    IPancakeClient.cs
└── Infraestructura/
    ├── Repositorio/  DocumentRepository.cs      ← MongoDB (UPSERT)
    └── Servicios/    PancakeClient.cs           ← HttpClient a pages.fm
```

---

## Colección `PancakePaginas`

Se guardan **todas** las páginas (activadas e inactivadas); campos en español:

```json
{
  "pageId": "850833678102371",
  "cuentaMadreId": "10001",
  "nombre": "Mi Página",
  "activada": true,
  "conectada": true,
  "pais": "CO",
  "tokenAccesoPagina": "eyJ...",
  "zonaHoraria": -5,
  "usuarios": [],
  "fechaCreacion": "2026-07-01",
  "fechaActualizacion": "2026-07-13"
}
```

**Clave de UPSERT:** `pageId`. Cada corrida refresca el estado (`activada`, `conectada`, token) sin duplicar.

---

## Variables de entorno

| Variable | Descripción | Valor ejemplo |
| -------- | ----------- | ------------- |
| `CADENA_CONEXION` | Cadena de conexión MongoDB (cifrada AES-256-ECB) | String cifrado |
| `DATABASE_NAME` | Nombre de la base de datos MongoDB | `"LogighoDB"` |

---

## Configuración Lambda

| Parámetro | Valor |
| --------- | ----- |
| Runtime | `dotnet8` |
| Handler | `ApiLambdaListarPaginasPancake::ApiLambdaListarPaginasPancake.Function::FunctionHandler` |
| Memory | `512 MB` |
| Timeout | `60 segundos` |
| Architecture | `x86_64` |

---

## Dependencias externas

| Servicio | Uso |
| -------- | --- |
| `Pancake (pages.fm)` | `GET /api/v1/pages` para listar las páginas de la cuenta |

---

## Historial de cambios

| Fecha | Autor | Cambio |
| ----- | ----- | ------ |
| 2026-07-13 | Iker Acevedo | Creación. Sincroniza páginas (activas e inactivas) a `PancakePaginas` con UPSERT por `pageId`. |

---

## Observaciones

- `HttpClient` es **estático** para evitar el agotamiento de sockets bajo concurrencia (patrón recomendado para Lambda).
- El `page_access_token` se lee de `settings.page_access_token` (anidado).
- La `zonaHoraria` puede venir "sucia" desde Pancake (ej. `7.0`); se modela como número.
- Se guardan también las páginas **inactivas** para tener trazabilidad completa; la lambda 4 se encarga de filtrar solo las activas con token.
