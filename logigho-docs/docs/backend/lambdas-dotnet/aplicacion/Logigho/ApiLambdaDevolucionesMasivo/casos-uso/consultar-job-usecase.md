## Autor: Iker Acevedo

Fecha creacion: 2026-08-26

Estado: produccion

## Caso de uso: ConsultarJobUseCase

**Ubicación:** `Aplicacion/CasosUso/ConsultarJobUseCase.cs`
**Lo invoca:** [`EstadoHandler`](../handlers/estado-handler.md)

---

## ¿Qué hace?

Traduce el documento de `JobsDevolucion` a la forma que consume el front. Es la consulta más frecuente del módulo (polling cada 2.5s), así que se limita a lo mínimo: un `findOne` por `_id` y el mapeo.

```
EjecutarAsync(jobId, emailUsuario, conDetalle, ct)
  job = JobRepository.ObtenerAsync(jobId)              si no existe -> NoEncontrado
  job.Usuario == emailUsuario?                          si no -> NoEncontrado (no NoAutorizado, ver abajo)
  devuelve Mapear(job, conDetalle)

ObtenerAbiertosAsync(emailUsuario, ct)
  JobRepository.ObtenerJobsAbiertosAsync(emailUsuario)   Estado in [Pendiente, EnProceso], limit 10
  mapea cada uno SIN detalle
```

---

## `Mapear(job, conDetalle)`

```csharp
procesadas = job.GuiasProcesadas                          // TotalResueltas + TotalYaProcesadas + TotalRechazadas
enCurso = job.Estado in [Pendiente, EnProceso]
latidoViejo = UtcNow - job.FechaUltimaActividad > 2 minutos
WorkerDetenido = enCurso && latidoViejo

Porcentaje = GuiasTotal == 0 ? 0 : round(procesadas * 100 / GuiasTotal)

Detalle = conDetalle ? job.Rechazadas : null
```

`WorkerDetenido` solo tiene sentido si el job sigue abierto — un job ya `Completado` o `Fallido` nunca lo reporta en `true`, aunque su último latido sea viejo (es normal, ya terminó).

---

## Por qué "no encontrado" y no "no autorizado" para un job ajeno

```csharp
if (!job.Usuario.Equals(emailUsuario, OrdinalIgnoreCase))
{
    Log.Warn(_ctx, "consulta-job-ajeno", ...);
    return NoEncontrado;   // nunca NoAutorizado
}
```

Responder "no autorizado" confirmaría que ese `jobId` **existe** pero pertenece a otra persona — información que el solicitante no debería poder deducir. Con "no encontrado", desde afuera es indistinguible un `jobId` inventado de uno real de otro usuario. Se logea igual con `Log.Warn` para tener trazabilidad interna del intento.

---

## Observaciones

- El detalle de guías **resueltas** nunca sale de acá — `job.Rechazadas` es el único array que el documento guarda completo (ver [ADR-002](../adr/ADR-002-jobdevolucion-solo-contadores.md)); las resueltas se consultan aparte, directo contra `InventarioDevolucion`.
- `ObtenerJobsAbiertosAsync` proyecta excluyendo `Rechazadas` — para reenganchar el polling alcanza con saber que el job existe y cómo va, no hace falta el detalle completo de rechazos de potencialmente varios jobs abiertos a la vez.
