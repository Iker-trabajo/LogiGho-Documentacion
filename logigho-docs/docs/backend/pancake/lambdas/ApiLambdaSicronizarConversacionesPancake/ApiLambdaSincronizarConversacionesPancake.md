## Autor: Iker Acevedo
Fecha creación: 2026-08-04

Estado: producción

## Lambda: ApiLambdaSincronizarConversacionesPancake

**Accionador:** Step Functions (`Task`, dentro de un `Map` — 1 invocación por página)

**AOT:** No

**Posición en el pipeline:** 2 de 3 ([ver Orquestación](../../conversaciones-mensajes/orquestacion.md))

---

## ¿Qué hace?

Trae **todas las conversaciones nuevas o actualizadas de 1 página** de Pancake (WhatsApp/Facebook) y las guarda en `PancakeConversaciones` (MySQL) con `UPSERT`. No trae mensajes — solo la metadata de la conversación (quién escribió, cuándo se actualizó, tipo de canal).

Usa un **watermark por página** (`MAX(FechaActualizacionPancake)` de lo ya guardado) para pedirle a Pancake solo lo que cambió desde la última corrida — nunca re-trae todo el historial completo en cada ejecución.

---

## Request

Recibe el `page_id` de la página a sincronizar (lo inyecta el `Map` del Step Function, 1 por iteración):

```json
{ "PageId": "waba_954021881132494" }
```

| Campo | Tipo | Requerido | Descripción |
|---|---|---|---|
| `PageId` | `string` | Sí | Id de la página en Pancake a sincronizar |

---

## Response

```json
{
  "PageId": "waba_954021881132494",
  "TotalSincronizadas": 62,
  "Error": false,
  "MensajeError": null
}
```

| Campo | Tipo | Descripción |
|---|---|---|
| `PageId` | `string` | Eco del page_id procesado |
| `TotalSincronizadas` | `int` | Cuántas conversaciones se insertaron/actualizaron en esta corrida |
| `Error` | `bool` | Si la corrida falló (el `Catch` del Step Function la ignora y sigue con las demás páginas) |
| `MensajeError` | `string?` | Detalle del error, si lo hubo |

---

## Flujo de datos

```mermaid
--8<-- "backend/pancake/conversaciones-mensajes/diagramas/flujo-sincronizar-conversaciones.mmd"
```

> Fuente de este diagrama: **[carpeta de diagramas](../../conversaciones-mensajes/diagramas/README.md)**.

## Flujo interno (detalle en código)

```
FunctionHandler (Function.cs)
  -> SincronizarConversacionesUseCase.EjecutarAsync(pageId, ahoraUtc)
       1. IPaginaTokenRepository.ObtenerAsync(pageId)      -> Mongo: token de pagina + token de usuario (cuenta madre)
       2. IConversacionRepository.ObtenerWaterMarkAsync    -> MySQL: MAX(FechaActualizacionPancake) ya guardado
       3. since = max(watermark, ahoraUtc - MAXIMO_DIAS_SINCRONIZAR)   // nunca trae mas historico que el tope
       4. Loop paginado (60 por pagina) contra Pancake:
            IPancakeConversacionesClient.ObtenerConversacionesAsync(pageId, token, since, until, cursor)
              -> si el token esta vencido (PancakeApiException): regenera con GenerarTokenPaginaAsync y reintenta
            -> ConversacionMapper.AEntidad(dto, pageId, ahoraUtc)  por cada conversacion
            -> IConversacionRepository.GuardarLoteAsync            -> MySQL: PancakeConversaciones (UPSERT por lote)
       5. Repite hasta que Pancake devuelve menos de 60 (fin de pagina) o vacio
  -> [RESULT] SalidaSincronizacion { PageId, TotalSincronizadas, Error }
```

---

## Arquitectura Clean Architecture

```
ApiLambdaSincronizarConversacionesPancake/
├── Function.cs
├── Dominio/
│   ├── Entidades/
│   │   └── Conversacion.cs          ← { ConversacionId, PageId, NumeroTelefono, NombreCliente, Tipo, Visto, FechaActualizacionPancake, DatosCrudos, FechaSincronizacion }
│   └── PancakeApiException.cs
├── Aplicacion/
│   ├── CasosUso/
│   │   └── SincronizarConversacionesUseCase.cs
│   ├── ConversacionMapper.cs         ← DTO Pancake -> Entidad de dominio
│   ├── DTO/
│   │   ├── ConversacionPancakeDto.cs
│   │   ├── EntradaSincronizacion.cs
│   │   └── SalidaSincronizacion.cs
│   └── Interfaces/
│       ├── IConversacionRepository.cs
│       └── IPancakeConversacionesClient.cs
└── Infraestructura/
    ├── Repositorio/
    │   ├── ConversacionRepository.cs   ← MySqlConnector, UPSERT por lote
    │   └── PaginaTokenRepository.cs    ← Mongo, token + regeneracion
    └── Servicios/
        └── PancakeConversacionesClient.cs  ← HttpClient contra Pancake
```

---

## Tabla destino (MySQL)

`PancakeConversaciones` — `UPSERT` (`ON DUPLICATE KEY UPDATE`) por `ConversacionId`:

| Columna | Origen |
|---|---|
| `ConversacionId`, `PageId` | Identidad de la conversación en Pancake |
| `NumeroTelefono` | `from.id` crudo de Pancake (`wa_57XXXXXXXXXX` para WhatsApp; PSID interno para Facebook — **no es un teléfono real** en ese caso) |
| `NombreCliente` | `from.name` |
| `Tipo` | Canal (`INBOX`, etc.) |
| `Visto` | `seen` de Pancake |
| `FechaActualizacionPancake` | `updated_at` de Pancake — es la fecha que se usa como watermark |
| `DatosCrudos` | JSON completo del DTO, por si hace falta reprocesar sin volver a pedirle a Pancake |
| `FechaSincronizacion` | Cuándo se guardó (hora Colombia) |
| `MensajesSincronizadosHasta` | **No la toca esta lambda** — la actualiza `ApiLambdaSincronizarMensajes` (ver esa doc) |
| `NumeroTelefonoNormalizado` | Columna **generada** por MySQL (`RIGHT(NumeroTelefono, 10)` solo si empieza con `wa_`) — para auditorías rápidas por número, ver [Operación](../../conversaciones-mensajes/operacion.md#vistas-sql-de-monitoreo) |

---

## Variables de entorno

| Variable | Descripción | Valor ejemplo |
|---|---|---|
| `CADENA_CONEXION` | Cadena de conexión MongoDB (cifrada AES-256-ECB) — para leer el token de la página | String cifrado |
| `DATABASE_NAME` | Base de datos MongoDB | `"LogighoDB"` |
| `CADENA_CONEXION_SQL` | Cadena de conexión MySQL (cifrada AES-256-ECB) | String cifrado |
| `MAXIMO_DIAS_SINCRONIZAR` | Tope de historia hacia atrás a traer (días) | `"5"` (default si falta) |

---

## Configuración Lambda

| Parámetro | Valor |
|---|---|
| Runtime | `dotnet10` |
| Handler | `ApiLambdaSincronizarConversacionesPancake::ApiLambdaSincronizarConversacionesPancake.Function::FunctionHandler` |
| Memory | `512 MB` |
| Timeout | `900 segundos` |
| Architecture | `x86_64` |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-08-04 | Iker Acevedo | Creación: sincroniza conversaciones de 1 página vía watermark, ventana original de 2 semanas. |
| 2026-08-06 | Iker Acevedo | Ventana bajada a `MAXIMO_DIAS_SINCRONIZAR=5` días (antes `MAXIMO_SEMANAS_SINCRONIZAR=2`) — el volumen histórico completo no era necesario para la operación diaria. |

---

## Observaciones

- El watermark (`ObtenerWaterMarkAsync`) hace que corridas repetidas sean baratas: si nada cambió en Pancake desde la última vez, `since` queda igual y la 1ª página de resultados vuelve vacía casi de inmediato.
- Esta lambda corre **1 página a la vez, secuencial internamente** (no tiene el paralelismo de `ApiLambdaSincronizarMensajes`) — el paralelismo entre páginas lo da el `Map` del Step Function (`MaxConcurrency: 8`), no esta lambda.
- Deuda pendiente: no persiste el token regenerado de vuelta a Mongo (a diferencia de `ApiLambdaSincronizarMensajes`, que sí lo hace) — como corre 1 sola invocación por página sin concurrencia interna, no sufre la condición de carrera que sí afectaba a la otra lambda, pero cada corrida nueva puede tener que regenerar el token de nuevo si Mongo quedó con el viejo.
