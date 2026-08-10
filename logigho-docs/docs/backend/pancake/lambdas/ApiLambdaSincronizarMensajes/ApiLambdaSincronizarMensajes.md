## Autor: Iker Acevedo
Fecha creación: 2026-08-04

Estado: producción

## Lambda: ApiLambdaSincronizarMensajes

**Accionador:** Step Functions (`Task`, invocada en loop directo — sin `Map`)

**AOT:** No

**Posición en el pipeline:** 3 de 3 ([ver Orquestación](../../conversaciones-mensajes/orquestacion.md))

---

## ¿Qué hace?

Toma un **lote** de conversaciones pendientes (de **cualquier** página, no de 1 sola) directo de MySQL, trae los mensajes de cada una **en paralelo**, y las guarda en `PancakeMensajes`. Al terminar cada conversación, marca su watermark (`MensajesSincronizadosHasta`) para que no se vuelva a procesar hasta que tenga actividad nueva.

Es la pieza que reemplazó al patrón "recibir la lista por el Step Function" — ver por qué en [Orquestación](../../conversaciones-mensajes/orquestacion.md#por-que-el-self-loop-y-no-un-map).

---

## Request

No recibe datos propios — se autoalimenta de MySQL en cada invocación:

```json
{}
```

---

## Response

```json
{
  "ConversacionesProcesadas": 800,
  "ConversacionesConError": 3,
  "TotalMensajesSincronizados": 4521,
  "SeCortoPorTiempo": false
}
```

| Campo | Tipo | Descripción |
|---|---|---|
| `ConversacionesProcesadas` | `int` | Cuántas del lote se intentaron (ok + error) — **este campo es lo que el `Choice` del Step Function mira para decidir si vuelve a invocar** |
| `ConversacionesConError` | `int` | Cuántas fallaron (token vencido sin recuperarse, error de Pancake, etc.) — quedan pendientes para la próxima vuelta |
| `TotalMensajesSincronizados` | `int` | Mensajes nuevos insertados en este lote |
| `SeCortoPorTiempo` | `bool` | Si la invocación se quedó sin tiempo (margen de 30s antes del timeout) antes de agotar el lote completo |

---

## Flujo de datos

```mermaid
--8<-- "backend/pancake/conversaciones-mensajes/diagramas/flujo-sincronizar-mensajes.mmd"
```

> Fuente de este diagrama: **[carpeta de diagramas](../../conversaciones-mensajes/diagramas/README.md)**.

## Flujo interno (detalle en código)

```
FunctionHandler (Function.cs)
  -> SincronizarMensajesPendientesUseCase.EjecutarAsync(limiteLote, tiempoRestante, ahoraUtc)
       1. IConversacionesPendientesRepository.ObtenerPendientesAsync(limiteLote)
            -> MySQL: SELECT ... WHERE MensajesSincronizadosHasta IS NULL
                         OR FechaActualizacionPancake > MensajesSincronizadosHasta
               ORDER BY FechaActualizacionPancake ASC LIMIT limiteLote
       2. IntercalarPorPagina(pendientes)
            -> reordena para que conversaciones consecutivas de la MISMA pagina no
               queden juntas (evita que 1 pagina monopolice el paralelismo, ver Observaciones)
       3. Parallel.ForEachAsync (GradoParalelismo = 80), por cada conversacion:
            SincronizarMensajesUseCase.EjecutarAsync(pageId, conversacionId, ahoraUtc)
              a. IPaginaTokenRepository.ObtenerAsync(pageId)         -> Mongo: token
              b. Loop paginado (30 mensajes/pagina) contra Pancake:
                   IPancakeMensajesClient.ObtenerMensajesAsync        -> rate-limit 3 req/s POR PAGINA (semaforo)
                     -> si token vencido: regenera coordinado por pagina (semaforo + cache), ver Observaciones
              c. Filtra mensajes dentro de MAXIMO_DIAS_SINCRONIZAR
              d. IMensajeRepository.GuardarLoteAsync                 -> MySQL: PancakeMensajes (UPSERT)
              e. IConversacionContextoRepository.ActualizarAsync     -> MySQL: PancakeConversaciones
                     .MensajesSincronizadosHasta = ahoraUtc  (SIEMPRE, tenga o no mensajes nuevos)
       4. Corta si tiemporestante < margen de seguridad (30s) -> SeCortoPorTiempo = true
  -> [RESULT] SalidaSincronizacionMensajesPendientes { ConversacionesProcesadas, ConversacionesConError, TotalMensajesSincronizados, SeCortoPorTiempo }
```

---

## Arquitectura Clean Architecture

```
ApiLambdaSincronizarMensajes/
├── Function.cs
├── Dominio/
│   ├── Entidades/
│   │   ├── Mensaje.cs                    ← { MensajeId, ConversacionId, Contenido, RemitenteId, RemitenteNombre, EsBot, RemitenteAdminNombre, RemitenteFlowId, FechaInsercion, DatosCrudos, FechaSincronizacion }
│   │   ├── ConversacionPendiente.cs      ← { ConversacionId, PageId }
│   │   └── ActualizacionContextoConversacion.cs
│   └── PancakeApiException.cs
├── Aplicacion/
│   ├── CasosUso/
│   │   ├── SincronizarMensajesPendientesUseCase.cs   ← orquesta el lote + paralelismo
│   │   └── SincronizarMensajesUseCase.cs             ← sincroniza 1 conversacion
│   ├── MensajeMapper.cs
│   ├── DTO/
│   │   ├── MensajePancakeDto.cs
│   │   ├── FechaUtcSinOffsetConverter.cs             ← fuerza UTC (Pancake no manda offset)
│   │   └── SalidaSincronizacionMensajesPendientes.cs / SalidaSincronizacionMensajes.cs
│   └── Interfaces/
│       ├── IMensajeRepository.cs
│       ├── IConversacionContextoRepository.cs
│       ├── IConversacionesPendientesRepository.cs
│       └── IPaginaTokenRepository.cs
└── Infraestructura/
    ├── Repositorio/
    │   ├── MensajeRepository.cs
    │   ├── ConversacionContextoRepository.cs
    │   ├── ConversacionesPendientesRepository.cs
    │   └── PaginaTokenRepository.cs
    └── Servicios/
        └── PancakeMensajesClient.cs       ← HttpClient + rate limiter por pagina
```

---

## Tablas involucradas (MySQL)

**`PancakeMensajes`** — `UPSERT` por `MensajeId`:

| Columna | Origen |
|---|---|
| `MensajeId`, `ConversacionId` | Identidad del mensaje |
| `Contenido` | Texto del mensaje |
| `RemitenteId`, `RemitenteNombre` | Quién lo mandó |
| `EsBot` | Si el remitente tiene `admin_name` (mensaje automatizado) |
| `RemitenteAdminNombre`, `RemitenteFlowId` | Metadata del bot/flujo, si aplica |
| `FechaInsercion` | Timestamp del mensaje en Pancake (UTC) |
| `DatosCrudos`, `FechaSincronizacion` | JSON completo + cuándo se guardó |

**`PancakeConversaciones`** — esta lambda actualiza (no inserta) las columnas de contexto:

| Columna | Cuándo se actualiza |
|---|---|
| `MensajesSincronizadosHasta` | **Siempre**, tenga o no mensajes nuevos la conversación — es lo que la saca de la cola de pendientes (ver Observaciones, bug histórico) |
| `UltimoRemitenteAdminId/Nombre`, `UltimoRemitentePlataforma`, `RemitenteFlowId`, `UltimoMensajeAutomatizado`, `ContextoCompletoCrudo` | Solo si hubo mensajes nuevos en esta corrida (si no, quedan `NULL`/sin tocar) |

---

## Variables de entorno

| Variable | Descripción | Valor ejemplo |
|---|---|---|
| `CADENA_CONEXION` | Cadena de conexión MongoDB (cifrada AES-256-ECB) | String cifrado |
| `DATABASE_NAME` | Base de datos MongoDB | `"LogighoDB"` |
| `CADENA_CONEXION_SQL` | Cadena de conexión MySQL (cifrada AES-256-ECB) | String cifrado |
| `MAXIMO_DIAS_SINCRONIZAR` | Tope de historia de mensajes por conversación (días) | `"5"` (default si falta) |
| `LIMITE_CONVERSACIONES_PENDIENTES` | Tamaño del lote que trae por invocación | `"800"` (default `200` si falta) |

---

## Configuración Lambda

| Parámetro | Valor |
|---|---|
| Runtime | `dotnet10` |
| Handler | `ApiLambdaSincronizarMensajes::ApiLambdaSincronizarMensajes.Function::FunctionHandler` |
| Memory | `2048 MB` |
| Timeout | `900 segundos` |
| Architecture | `x86_64` |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-08-04 | Iker Acevedo | Creación: recibía la lista de conversaciones por el Step Function. Falló en producción con `States.DataLimitExceeded` (1.3M conversaciones no caben en el estado de 256KB). |
| 2026-08-05 | Iker Acevedo | Rediseño completo: autoconsulta MySQL, self-loop, `Parallel.ForEachAsync` (paralelismo 40→80), intercalado por página. Se corrige bug de loop infinito: el watermark solo se marcaba si había mensajes nuevos, dejando conversaciones sin actividad reciente reprocesándose para siempre. |
| 2026-08-06 | Iker Acevedo | Se corrige condición de carrera en la regeneración de tokens: múltiples conversaciones paralelas de la misma página regeneraban el token por separado y se invalidaban entre sí (`error_code=105` en ~90% de un lote). Ahora coordinado por semáforo + caché a nivel de instancia, y persistido de vuelta a Mongo. Ventana bajada de 2 semanas a 5 días. Memoria subida a 2048MB. |

---

## Observaciones

- **Bug histórico — loop infinito por watermark parcial** (corregido): si `SincronizarMensajesUseCase` no encontraba mensajes nuevos, no llamaba a `ActualizarAsync`, así que `MensajesSincronizadosHasta` quedaba `NULL` para siempre — esa conversación volvía a salir "pendiente" en cada corrida sin parar. Fix: `ActualizarAsync` se llama siempre, con los campos de contexto en `NULL` si no hubo mensajes.
- **Bug histórico — condición de carrera en tokens** (corregido): con paralelismo alto, si 30 conversaciones de la misma página detectaban el token vencido al mismo tiempo, las 30 pedían un token nuevo por separado — Pancake invalida el token anterior al emitir uno nuevo, así que la mayoría terminaba fallando igual. Fix: semáforo por página (`SemaphoreSlim`) + caché del token fresco a **nivel de instancia** de `SincronizarMensajesUseCase` (no `static` — `Function.cs` ya crea 1 sola instancia compartida entre las 80 tareas paralelas de una invocación, así que alcanza sin filtrar estado entre invocaciones o tests).
- **Por qué el intercalado por página**: `Parallel.ForEachAsync` llena sus slots en el orden de la lista. Si la consulta SQL (`ORDER BY FechaActualizacionPancake ASC`) devuelve muchas filas seguidas de la misma página, los 80 slots paralelos quedan todos esperando el mismo semáforo de rate-limit de esa página — el paralelismo no sirve de nada. `IntercalarPorPagina` reordena en round-robin para que conversaciones consecutivas sean de páginas distintas.
- **Rate limit**: 3 req/s por página (`PancakeMensajesClient`), con margen bajo el límite real de Pancake (5 req/s) — deliberado, no ajustar sin motivo.
