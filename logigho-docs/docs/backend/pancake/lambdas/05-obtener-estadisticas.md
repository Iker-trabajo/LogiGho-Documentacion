## Autor: Iker Acevedo
Fecha creacion: 2026-07-13

Estado: produccion

## Lambda: ApiLambdaObtenerEstadisticas

**Accionador:** Step Functions (dentro del **2º `Map`** — una iteración por página, MaxConcurrency 5)

**AOT:** No

**Posición en el pipeline:** 5 de 5

---

## ¿Qué hace?

Para **una página**, consulta a Pancake las estadísticas por campaña dentro de la ventana de tiempo calculada, y las guarda con **doble escritura**: en **MongoDB** y en la **RDS Aurora MySQL** del almacén de datos. Un registro por campaña, por franja del día.

Llamada a Pancake:

```
GET https://pages.fm/api/public_api/v1/pages/{page_id}/statistics/pages_campaigns
      ?page_access_token={token}&since={since}&until={until}
```

---

## Request

Cada iteración del 2º `Map` recibe **una página + el contexto de la ventana**, combinados por el `ItemSelector` del Step Function:

```json
{
  "page_id": "850833678102371",
  "page_access_token": "eyJhbGciOiJ...",
  "timezone": -5,
  "since": 1783832400,
  "until": 1783918800,
  "fecha_reporte": "2026-07-12",
  "tipo": "cierre_dia_anterior",
  "slot_id": "1"
}
```

| Campo | Tipo | Requerido | Descripción |
| ----- | ---- | --------- | ----------- |
| `page_id` | `string` | Sí | Página a consultar |
| `page_access_token` | `string` | Sí | Token de la página |
| `since` / `until` | `long` | Sí | Ventana en Unix UTC (de la lambda 1) |
| `fecha_reporte` | `string` | Sí | Fecha del reporte (se guarda en cada registro) |
| `tipo` | `string` | Sí | `cierre_dia_anterior` / `intradia_actual` |
| `slot_id` | `string` | Sí | Franja de la corrida (se guarda en cada registro) |
| `timezone` | `number` | No | Zona horaria de la página |

---

## Response

```json
{ "page_id": "850833678102371", "status": "ok", "campanias": 2 }
```

| `status` | Significado |
| -------- | ----------- |
| `ok` | Se consultó y guardó correctamente (0 o más campañas) |
| `error` | Pancake rechazó la consulta (ej. token vencido). Se guarda una fila-centinela con el detalle |
| `omitido` | `page_id` o token vacío — no se hizo la llamada |

!!! note "200 OK ≠ éxito"
    Pancake puede responder **HTTP 200 con `success:false`** en el cuerpo (ej. token inválido, `error_code:102`). El cliente revisa `success` y lanza `PancakeApiException` → se guarda `status:"error"`, sin tumbar la corrida de las demás páginas.

---

## Doble escritura (Mongo + Aurora MySQL)

El guardado usa el **patrón Composite**: un repositorio que envuelve a los dos.

```
CompositeEstadisticasRepository
  ├── MongoEstadisticasRepository   (PRIMARIO)   → si falla, propaga → el Catch del Map marca la iteración
  └── SqlEstadisticasRepository     (BEST-EFFORT) → si falla, se loguea y NO bloquea (Mongo ya quedó)
```

- **Idempotencia:** UPSERT por la clave `(pageId, campañaId, fechaReporte, slotId)`. Volver a correr la misma franja **actualiza**, no duplica.
- **Plug-and-play (SQL):** el repositorio SQL arranca **inerte** si no existe la variable `CADENA_CONEXION_SQL`; en cuanto se configura, se activa solo sin tocar código (ver **[Operación → Doble escritura RDS](../operacion.md#doble-escritura-rds)**).
- Los valores se guardan **como string, tal cual llegan de Pancake**. El campo de campaña se llama `campañaId` (con ñ) en Mongo.

---

## Flujo interno

```
FunctionHandler (Function.cs)  [Map: una página]
  -> Valida page_id y token (si vacío -> status "omitido")
  -> ObtenerEstadisticasUseCase.EjecutarAsync(entrada)
       -> IEstadisticasPancakeClient.ObtenerCampaniasAsync(page_id, token, since, until)
            -> HttpClient estático GET .../statistics/pages_campaigns?... (URL nunca logueada)
            -> Si !success -> PancakeApiException
            -> Mapea la respuesta a List<Campaña>
       -> CompositeEstadisticasRepository.GuardarAsync(ctx, campañas, errorDetail)
            -> Mongo  (UPSERT por clave)          [primario]
            -> MySQL  (INSERT ... ON DUPLICATE KEY UPDATE, en transacción)  [best-effort]
  -> [RESULT] { page_id, status, campanias }
```

---

## Arquitectura Clean Architecture

```
ApiLambdaObtenerEstadisticas/
├── Function.cs                              ← Composition Root (Mongo y SQL cacheados estáticos)
├── Dominio/
│   └── Entidades/
│       └── Campaña.cs  /  ContextoReporte
├── Aplicacion/
│   ├── CasosUso/ ObtenerEstadisticasUseCase.cs
│   ├── DTO/ EntradaEstadisticas.cs · SalidaEstadisticas.cs
│   └── Interfaces/
│       ├── Repositorios/ IEstadisticasRepository.cs
│       └── Servicios/    IEstadisticasPancakeClient.cs
└── Infraestructura/
    ├── Repositorio/
    │   ├── MongoEstadisticasRepository.cs       ← Mongo (primario)
    │   ├── SqlEstadisticasRepository.cs         ← Aurora MySQL (best-effort)
    │   ├── CompositeEstadisticasRepository.cs   ← Combina ambos
    │   ├── EstadisticaPaginaDocument.cs         ← Modelo de persistencia Mongo
    │   └── PancakeEstadisticasPaginas.sql       ← DDL propuesto de la tabla MySQL
    └── Servicios/
        ├── EstadisticasPancakeClient.cs
        └── Pancake/ PancakeStatsDtos.cs · PancakeApiException.cs
```

---

## Salida en `PancakeEstadisticasPaginas` (un registro por campaña)

```json
{
  "pageId": "850833678102371",
  "campañaId": "23851234567890",
  "fechaReporte": "2026-07-12",
  "slotId": "1",
  "tipoCalculo": "cierre_dia_anterior",
  "since": 1783832400,
  "until": 1783918800,
  "status": "ok",
  "nombre": "Campaña Julio",
  "moneda": "COP",
  "gasto": "125000",
  "cpc": "350",
  "cpm": "8200",
  "ctr": "1.8",
  "alcance": "34000",
  "clics": "420",
  "impresiones": "62000",
  "resultados": "37",
  "fechaActualizacion": "2026-07-13T04:36:07Z"
}
```

La misma estructura se replica en la tabla MySQL `PancakeEstadisticasPaginas` (ver el DDL en el proyecto).

---

## Variables de entorno

| Variable | Descripción | Valor ejemplo |
| -------- | ----------- | ------------- |
| `CADENA_CONEXION` | Cadena MongoDB (cifrada AES-256-ECB) | String cifrado |
| `DATABASE_NAME` | Base de datos MongoDB | `"LogighoDB"` |
| `CADENA_CONEXION_SQL` | Cadena Aurora MySQL (cifrada AES-256-ECB). **Si falta, SQL queda inerte** | String cifrado |
| `TABLA_ESTADISTICAS_SQL` | *(opcional)* nombre de la tabla destino si difiere | `"PancakeEstadisticasPaginas"` |

---

## Configuración Lambda

| Parámetro | Valor |
| --------- | ----- |
| Runtime | `dotnet8` |
| Handler | `ApiLambdaObtenerEstadisticas::ApiLambdaObtenerEstadisticas.Function::FunctionHandler` |
| Memory | `512 MB` |
| Timeout | `60 segundos` |
| Architecture | `x86_64` |

---

## Dependencias externas

| Servicio | Uso |
| -------- | --- |
| `Pancake (pages.fm)` | `GET /statistics/pages_campaigns` para traer las campañas |
| `RDS Aurora MySQL` | Segunda copia de las estadísticas (doble escritura, best-effort) |

---

## Historial de cambios

| Fecha | Autor | Cambio |
| ----- | ----- | ------ |
| 2026-07-13 | Iker Acevedo | Creación. Recolección de estadísticas por campaña + escritura en MongoDB. Manejo de `200 OK` con `success:false`. |
| 2026-07-13 | Iker Acevedo | Doble escritura a Aurora MySQL con patrón Composite (Mongo primario, SQL best-effort), `MySqlConnector`, UPSERT idempotente. Modo inerte si falta `CADENA_CONEXION_SQL`. Validado end-to-end contra la RDS de producción. |

---

## Observaciones

- **Best-effort en SQL:** si la RDS falla, se loguea `[Composite] Falló SQL (no bloquea)` y el pipeline continúa — MongoDB nunca se ve afectado.
- **Diagnóstico rápido en CloudWatch:** el log `[SqlRepo] RDS configurada -> ... HABILITADA` confirma que la doble escritura está activa; la ausencia de `[Composite] Falló SQL` confirma que SQL escribió bien.
- **Concurrencia:** el 2º Map corre con `MaxConcurrency 5` y `Retry` con backoff exponencial ante `Lambda.TooManyRequestsException` (throttling). Si se sube la concurrencia, considerar **RDS Proxy** para el pooling de conexiones.
- **Seguridad pendiente (infra):** endurecer SSL a `VerifyCA` y cerrar el Security Group de la RDS a solo la VPC/IPs necesarias.
