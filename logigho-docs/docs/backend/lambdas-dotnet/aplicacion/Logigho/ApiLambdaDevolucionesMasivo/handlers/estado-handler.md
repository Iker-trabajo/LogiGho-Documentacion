## Autor: Iker Acevedo

Fecha creacion: 2026-08-26

Estado: produccion

## Handler: EstadoHandler

**Ruta:** `GET /devolucionesMasivo/estado`
**Ubicación:** `Handlers/EstadoHandler.cs`
**Timeout:** 30s — corre detrás de API Gateway

---

## ¿Qué hace?

La ruta más llamada de todo el módulo: el front la consulta cada **2.5 segundos** por cada operario con un lote activo (polling). Por eso está diseñada para ser barata — un `findOne` por `_id` y nada más, sin joins ni agregaciones.

Tiene 3 comportamientos según los query params:

| Query | Comportamiento |
| ----- | -------------- |
| sin `jobId` | Jobs **abiertos** (`Pendiente`/`EnProceso`) del usuario — para reenganchar el polling si recargó la página |
| `?jobId=X` | Estado de un job puntual, sin el detalle de rechazadas |
| `?jobId=X&detalle=true` | Igual, más el array `Rechazadas` completo — se pide **una sola vez**, cuando el job termina |

---

## Response — 200 OK (con `jobId`)

```json
{
  "Error": false,
  "JobId": "a3f9c1e2...",
  "Estado": "EnProceso",
  "GuiasTotal": 812,
  "GuiasProcesadas": 350,
  "GuiasPendientes": 462,
  "Resueltas": 300,
  "Rechazadas": 20,
  "YaProcesadas": 30,
  "Porcentaje": 43,
  "FechaInicio": "2026-08-26T00:24:32Z",
  "FechaUltimaActividad": "2026-08-26T00:31:02Z",
  "FechaFin": null,
  "WorkerDetenido": false,
  "MensajeError": null,
  "Detalle": null
}
```

| Código | Cuándo |
| ------ | ------ |
| `401` | Token inválido, o usuario sin tiendas asignadas |
| `404` | El `jobId` no existe, **o existe pero pertenece a otro usuario** (ver seguridad más abajo) |
| `500` | Excepción no controlada |

---

## Flujo interno

```
EjecutarAsync(peticion, contexto)
  identidad = LectorTokenCognito.Leer(headers)
  usuario = UsuarioRepository.ObtenerContextoAsync(identidad)   el email REAL sale de la BD, no del token

  jobId = query["jobId"]

  si no hay jobId:
    ConsultarJobUseCase.ObtenerAbiertosAsync(usuario.Email)
      JobRepository.ObtenerJobsAbiertosAsync   filtro Estado in [Pendiente, EnProceso], limit 10
    responde 200 con la lista

  si hay jobId:
    conDetalle = query["detalle"] == "true"
    ConsultarJobUseCase.EjecutarAsync(jobId, usuario.Email, conDetalle)
      JobRepository.ObtenerAsync(jobId)
      valida job.Usuario == usuario.Email        (ver seguridad)
      Mapear(job, conDetalle)                     arma la respuesta, incluye WorkerDetenido
    responde segun EstadoConsulta: Ok / NoEncontrado / NoAutorizado
```

---

## `WorkerDetenido` — cómo se calcula

```csharp
ToleranciaLatido = 2 minutos
enCurso = job.Estado in [Pendiente, EnProceso]
latidoViejo = DateTime.UtcNow - job.FechaUltimaActividad > ToleranciaLatido
WorkerDetenido = enCurso && latidoViejo
```

El Worker actualiza `FechaUltimaActividad` en **cada lote** (`ActualizarProgresoAsync`). Dos minutos de silencio son mucho más de lo que tarda cualquier lote normal — si pasan y el job sigue abierto, se asume que el Worker murió sin completar ni fallar explícitamente (por ejemplo, un cold start que agotó las 3 continuaciones sin que el código llegara a marcar `Fallido`, o un corte de red total).

---

## Seguridad: un job solo lo puede ver su dueño

```csharp
if (!job.Usuario.Equals(emailUsuario, OrdinalIgnoreCase))
    return NoEncontrado;  // NO NoAutorizado
```

Se responde **"no encontrado"** y no **"no autorizado"** a propósito: decir "existe pero no es tuyo" ya confirma que ese `jobId` existe, información que el solicitante no debería tener. Mismo criterio que usa GitHub con los repositorios privados — un 404 no distingue entre "no existe" y "no tienes permiso".

---

## Observaciones

- El `Detalle` (array `Rechazadas`) solo se pide con `detalle=true`, y el front lo pide una sola vez al terminar el job — pedirlo en cada poll con 3.000 guías serían megabytes por consulta cada 2.5 segundos.
- El detalle de las guías **resueltas** nunca viaja por esta ruta — vive en `InventarioDevolucion`, consultado directo por el front vía `metodoGenerico` (ver [`ingreso-devoluciones-repository`](../../../../../frontend/views/logistica/ingreso-devoluciones/servicios/ingreso-devoluciones-repository.md)).
- Ver [`ConsultarJobUseCase`](../casos-uso/consultar-job-usecase.md) para el mapeo completo del documento a la respuesta.
