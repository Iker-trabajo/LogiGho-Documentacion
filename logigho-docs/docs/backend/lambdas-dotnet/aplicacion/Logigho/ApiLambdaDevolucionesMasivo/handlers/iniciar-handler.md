## Autor: Iker Acevedo

Fecha creacion: 2026-08-26

Estado: produccion

## Handler: IniciarHandler

**Ruta:** `POST /devolucionesMasivo/iniciar`
**Ubicación:** `Handlers/IniciarHandler.cs`
**Timeout:** 30s — corre detrás de API Gateway

---

## ¿Qué hace?

Punto de entrada del módulo. Valida al usuario, crea el documento del job en `JobsDevolucion`, dispara al [`WorkerHandler`](worker-handler.md) de forma asíncrona, y responde en ~200ms — sin tocar ninguna transportadora ni `PedidosInter`. Todo lo pesado lo hace el Worker después, sin que el front tenga que esperarlo.

---

## Request

```json
{
  "Guias": ["240012345678", "990087654321", "2294999001"]
}
```

## Response — 200 OK

```json
{
  "Error": false,
  "Mensaje": "Proceso iniciado.",
  "JobId": "a3f9c1e2d4b5467a9c1e2d4b5467a9c1",
  "GuiasAceptadas": 812,
  "GuiasDescartadas": 14
}
```

`GuiasDescartadas` cuenta las vacías o repetidas **dentro del mismo archivo** (deduplicadas antes de crear el job), no las que luego fallen al procesarse.

| Código | Cuándo |
| ------ | ------ |
| `400` | Cuerpo vacío/inválido, cero guías recibidas, ninguna guía con formato válido tras limpiar, o supera el tope de 3.000 (`JobDevolucion.MaximoGuiasPorJob`) |
| `401` | Token inválido, o usuario sin tiendas asignadas en la colección `Users` |
| `500` | Excepción no controlada |

---

## Flujo interno

```
EjecutarAsync(peticion, contexto)
  LectorTokenCognito.Leer(peticion.Headers)     header "Token", no "Authorization"
  Deserializa el body a IniciarJobRequest
  IniciarJobUseCase.EjecutarAsync(cuerpo, identidad)
    valida identidad.EsValida
    UsuarioRepository.ObtenerContextoAsync(identidad)   por email o por sub/cognitoId
    NormalizadorGuia.Limpiar() + Distinct() por cada guia recibida
    valida guias.Count > 0 y <= MaximoGuiasPorJob (3000)
    crea JobDevolucion { JobId = Guid nuevo, Estado = Pendiente }
    JobRepository.CrearAsync(job)
  try { DispararWorkerAsync(jobId) }
    Lambda.InvokeAsync({ FunctionName: NOMBRE_LAMBDA_WORKER, InvocationType: Event, Payload: {jobId} })
    catch -> solo logea (el job ya existe en Mongo; perder la respuesta perderia el JobId sin forma de recuperarlo)
  responde 200 con JobId, GuiasAceptadas, GuiasDescartadas
```

`DispararWorkerAsync` usa `InvocationType.Event`: AWS encola la invocación y devuelve el control de inmediato, sin esperar a que el Worker termine. Es la única forma de "disparar y olvidar" sin quedar atado al techo de 30 segundos del gateway.

---

## Observaciones

- Si `DispararWorkerAsync` falla (por ejemplo, `NOMBRE_LAMBDA_WORKER` no configurada), el error se traga y solo se logea — el job **ya quedó creado** en Mongo, y perder la respuesta HTTP en ese punto dejaría al operario sin el `JobId` para hacer seguimiento, sin ninguna forma de recuperarlo.
- El tope de 3.000 guías se valida acá y también en el front (`carga-archivo.component.ts`) — el front no es una barrera real, un cliente podría llamar al endpoint directo, así que la validación de backend es la que cuenta.
- Ver [`IniciarJobUseCase`](../casos-uso/iniciar-job-usecase.md) para el detalle de la validación.
