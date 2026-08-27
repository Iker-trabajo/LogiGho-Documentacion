## Autor: Iker Acevedo

Fecha creacion: 2026-08-26

Estado: produccion

## Caso de uso: ProcesarJobUseCase

**Ubicación:** `Aplicacion/CasosUso/ProcesarJobUseCase.cs`
**Lo invoca:** [`WorkerHandler`](../handlers/worker-handler.md)

---

## ¿Qué hace?

El corazón del módulo. Procesa las guías pendientes de un job en lotes de 200, siguiendo un orden de resolución en 4 fases que minimiza el trabajo contra Mongo y contra las APIs externas: lo barato primero, lo caro solo para lo que sobra.

```
FASE A  consultas en lote a Mongo    (resuelve la mayoria, barato)
FASE B  estrategias por formato      (solo lo que Mongo no pudo)
FASE C  reintento con otra           (solo lo que fallo por mal ruteo)
FASE D  validaciones transversales   (anulado, permiso de tienda)
ESCRITURA en lote + inventario en una sola llamada
```

---

## `EjecutarAsync(jobId, ct)` — el bucle principal

```
job = JobRepository.ObtenerAsync(jobId)
usuario = UsuarioRepository.ObtenerContextoAsync(job.Usuario)   se lee UNA vez por invocacion, no por guia

pendientes = Queue<string>(job.GuiasPendientes)

mientras pendientes.Count > 0:
  si MargenParaOtroLote != null && RemainingTime < margen:
    return PedirContinuacionAsync(...)     no arranca un lote a medias

  lote = toma hasta 200 de pendientes

  resultados = ProcesarLoteAsync(lote, usuario, jobId, ct)     las 4 fases, ver abajo

  rechazadas = resultados donde Estado == Rechazada
  yaProcesadasCount = resultados donde Estado == YaProcesada
  resueltasCount = resto

  JobRepository.ActualizarProgresoAsync(
      jobId, rechazadas, resueltasCount, yaProcesadasCount, pendientes.ToList())

JobRepository.MarcarCompletadoAsync(jobId)
```

`TamanoLote = 200`: acota cuánto se pierde si el Worker muere a mitad de un lote (nunca más de 200 guías sin persistir), y mantiene el tamaño del `$push` sobre `Rechazadas` razonable.

---

## `ProcesarLoteAsync` — las 4 fases en detalle

```
FASE A.1 — Idempotencia (lo mas barato, primero)
  ObtenerGuiasYaProcesadasAsync(lote)
    filtra InventarioDevolucion por Guia + Validacion=="OK"
  -> las que YA estan: Estado = YaProcesada, no tocan Mongo de nuevo ni ninguna API
  -> el resto sigue a A.2

FASE A.2 — Match directo en PedidosInter
  BuscarPorGuiaDevolucionAsync(porResolver)      Inter ya enlazada en corrida anterior
  BuscarPorNumeropreenvioAsync(porResolver)      Servientrega (su guia devolucion ES el preenvio)
  -> las encontradas: Estado = ResueltaDirecta
  -> el resto sigue a B

FASE B — Estrategias por formato
  FactoryEstrategiaDevolucion.AgruparLote(porResolver)
    reparte cada guia a la estrategia que "PuedeResolver" su formato
    (Inter: 13 digitos empieza en "30"; Envia: 12 digitos)
  por cada grupo: EstrategiaConsultaExterna.ResolverLoteAsync
    -> ver infraestructura/clientes-transportadora.md
  las que "MereceReintentoConOtraTransportadora" (SinGuiaOriginalEnRespuesta)
  se apartan para la FASE C, no se agregan a resultados todavia

FASE C — Reintento con otra transportadora
  Solo para las que ninguna estrategia reconocio bien el formato.
  Se agrupan por la transportadora que ya fallo y se prueban las ALTERNATIVAS
  (FactoryEstrategiaDevolucion.ObtenerAlternativas)
  Las que ninguna alternativa resuelve, conservan su rechazo original.

FASE D — Reglas transversales (AplicarReglas)
  Solo sobre las que llegaron a un pedido concreto.
  Por cada regla en orden (ReglaPermisoTienda, ReglaPedidoAnulado):
    si Validar() devuelve un motivo -> se rechaza AHI, no sigue a la siguiente regla
  Las reglas corren DESPUES de resolver: primero se sabe de que pedido se trata,
  despues si corresponde tocarlo.

ESCRITURA (PersistirAsync)
  1. MarcarDevolucionCompletadaAsync   solo las EsRecienResuelta (no las YaProcesada)
  2. RegistrarEnInventarioDevolucionAsync   TODAS menos las YaProcesada, con el JobId
  3. ActualizarLoteAsync (inventario)   solo las RequiereActualizarInventario
     — este orden es obligatorio: el paso 3 lee el pedido desde Mongo y decide que
       movimiento registrar segun su Estado; si el paso 1 no corrio antes, no
       encuentra nada que actualizar.
```

---

## Orden fijo de las reglas de validación

```csharp
_reglas = reglas.OrderBy(r => r.Orden).ToList();
```

Sin este orden explícito, el motivo de rechazo dependería de cómo el sistema de inyección de dependencias devolviera las reglas — que podría cambiar entre despliegues sin que nadie toque el código. `ReglaPermisoTienda.Orden = 10`, `ReglaPedidoAnulado` corre después.

---

## Reintento con otra transportadora

Solo aplica al motivo `SinGuiaOriginalEnRespuesta` — la transportadora respondió, pero no había ningún número extraíble en el texto. Eso sugiere que la guía fue enrutada a la transportadora equivocada por su formato (por ejemplo, un caso límite donde el largo/prefijo coincide con el patrón de otra). El reintento agrupa por transportadora ya intentada y prueba las alternativas una por una, hasta que alguna resuelva o se agoten.

**No** reintenta `ErrorConsultaExterna` (falla técnica) con otra transportadora — reintentar contra otra API no resuelve un problema de red o de disponibilidad; ese motivo se marca como reintentable **desde el front**, con un nuevo lote más adelante.

---

## Manejo de fallos de inventario

```csharp
if (!res.Exitoso)
    Log.Error(...);   // NO se relanza, NO tumba el job
```

Un fallo actualizando inventario no revierte la devolución ya registrada — el pedido sigue siendo elegible en una corrida posterior de `ApiLambdaActualizacionInventario`, porque esa lambda filtra por su propia marca de "ya procesado". Tumbar el job entero por un fallo de inventario sería mucho peor que dejar ese movimiento pendiente para la próxima corrida.

---

## Observaciones

- `ReintentarAsync` conserva el rechazo original (con su motivo) para las guías que ninguna alternativa pudo resolver — no las marca con un motivo genérico de "reintento fallido".
- `Log.Warn(_ctx, "guia-rechazada-por-regla", ...)` registra **los dos lados** de la comparación (tienda del pedido vs. tiendas del usuario) — si algún día un permiso falla por una diferencia invisible (un acento, un espacio), el log lo hace evidente en CloudWatch sin tener que reproducir el caso.
- Ver [ADR-002](../adr/ADR-002-jobdevolucion-solo-contadores.md) para por qué solo `Rechazadas` se acumula en el documento del job, y las resueltas solo suman a un contador.
