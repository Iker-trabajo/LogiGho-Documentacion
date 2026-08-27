## Autor: Iker Acevedo

Fecha creacion: 2026-08-26

Estado: produccion

## Caso de uso: IniciarJobUseCase

**Ubicación:** `Aplicacion/CasosUso/IniciarJobUseCase.cs`
**Lo invoca:** [`IniciarHandler`](../handlers/iniciar-handler.md)

---

## ¿Qué hace?

Valida la petición y crea el job. Nunca toca `PedidosInter` ni ninguna API de transportadora — todo lo pesado queda para el Worker.

```
EjecutarAsync(peticion, identidad, ct)
  1. identidad.EsValida?                                    si no -> NoAutorizado
  2. UsuarioRepository.ObtenerContextoAsync(identidad)       si no existe / sin tiendas -> NoAutorizado
  3. peticion.Guias vacio?                                   -> PeticionInvalida
  4. NormalizadorGuia.Limpiar() + Distinct() sobre cada guia
     si queda vacio tras limpiar                             -> PeticionInvalida
  5. guias.Count > MaximoGuiasPorJob (3000)                  -> PeticionInvalida
  6. crea JobDevolucion { JobId = Guid.NewGuid("N"), Estado = Pendiente }
     JobRepository.CrearAsync(job)
  7. devuelve Ok(job.JobId, guias.Count, descartadas)
```

El orden importa: se valida identidad **antes** de mirar las guías — si no se sabe quién es, no tiene sentido validar nada más.

---

## Por qué se limpia y deduplica antes de crear el job

```csharp
var guias = peticion.Guias
    .Select(NormalizadorGuia.Limpiar)
    .Where(g => g.Length > 0)
    .Distinct(StringComparer.OrdinalIgnoreCase)
    .ToList();
```

Sin esto, dos problemas reales: procesar la misma guía dos veces dentro del mismo job (gastando el doble de llamadas a la API externa para nada), y que el conteo que ve el operario en el front (`GuiasAceptadas`) no cuadre con lo que el backend realmente terminó procesando.

`GuiasDescartadas = recibidas - aceptadas` — cuántas venían vacías o repetidas **dentro del mismo archivo**. No tiene nada que ver con cuántas después se rechazan al procesar (eso es otro contador, `TotalRechazadas`, que vive en el job).

---

## Por qué un usuario sin tiendas se corta acá

```csharp
if (usuario is null)
    return NoAutorizado("El usuario no existe o no tiene tiendas asignadas.");
```

Sin este corte temprano, un usuario sin tiendas asignadas terminaría con las 3.000 guías rechazadas una por una con `SinPermisoTienda`, después de gastar el trabajo completo de resolverlas contra Mongo y las APIs externas. Cortar acá, antes de crear el job, ahorra ese trabajo completo y le da al operario un mensaje que se entiende de inmediato.

---

## Observaciones

- El tope de 3.000 se repite acá aunque el front ya lo valide — el front no es una barrera de seguridad real, el endpoint se puede llamar directo.
- `Log.Info(_ctx, "iniciar-job", ...)` registra recibidas/aceptadas/descartadas — es el log que permite auditar si un archivo "raro" (muchas guías vacías) generó un job mucho más chico de lo esperado.
