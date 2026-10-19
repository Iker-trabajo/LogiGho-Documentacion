## Autor: Iker Acevedo

Fecha creacion: 2026-10-18  
Estado: desarrollo

---

# Caso de uso: SolicitarNovedadesUseCase

`Aplicacion/CasosUso/SolicitarNovedadesUseCase.cs` — ruta `/novedades-inter/solicitar`.

Decide **qué guías** consultar a ws-inter en cada corrida y las envía. Nunca espera el resultado: ws-inter responde `202`
y entrega la respuesta después por el webhook ([Recibir resultado](recibir-resultado-usecase.md)).

## Qué guías cuentan como "novedad"

`ReglaSeleccionNovedad`, configurable por variables de entorno. Una guía de `PedidosInter` (transportadora
`INTERRAPIDISIMO`, estado no final) está en novedad si:

- su `Estado` es uno de: *Intento de entrega, Telemercadeo, Reenvio, Devolucion Regional, Reclame en oficina*; **o**
- su `Descripcion Asociada` es *No logramos hacer la entrega* o *Tu envío presenta una novedad*.

| Prioridad | Cuándo | Refresco |
| --------- | ------ | -------- |
| **Alta** | La descripción indica fallo de entrega | cada 6 h |
| **Normal** | El resto (sobre todo *Reclame en oficina*, que dice *Ya puedes recoger tu envío*) | cada 24 h |

Estados finales excluidos (los mismos que la lambda legacy): Pagada, Archivada, Devolución ratificada, Anulada,
Indemnizacion, Devolucion Completada.

## Cuándo se (re)envía una guía — `PlanificadorEnvios`

| Situación | Acción |
| --------- | ------ |
| Sin seguimiento | Enviar |
| Seguimiento `Cerrado` (la guía volvió a novedad) | Enviar y reactivar |
| `Enviada`, sin respuesta, hace menos de 2 h | Saltar (se espera el callback) |
| `Enviada`, sin respuesta, 2 h o más, menos de 3 envíos | Enviar (reintento) |
| `Enviada`, sin respuesta, 2 h o más, 3 envíos ya | Marcar `Fallida` y alertar |
| Cambió `Estado` o `Descripcion Asociada` en Inter | Enviar |
| `Fallida` y venció el refresco | Enviar |
| `Respondida`/`NoEncontrada` y venció el refresco | Enviar |
| Cualquier otra | Saltar |

Cuando una guía sale de novedad su seguimiento pasa a `Cerrado` y **no se consulta más**.

## Orden de una corrida

1. `GET /health` de ws-inter. Si falla: alerta de Slack, `wsInterNoDisponible=true` y se termina.
2. Lee candidatos de `PedidosInter` y sus seguimientos.
3. Cierra los seguimientos activos que ya no están en la lista.
4. **Reproyección**: si un seguimiento tiene motivo pero `PedidosInter.MotivoNovedadInter` no coincide (la lambda legacy
   reescribió el pedido), lo vuelve a escribir.
5. Planifica y ordena: prioridad Alta primero, luego las que hace más tiempo no se consultan. Aplica `MAX_GUIAS_POR_CORRIDA`.
6. Envía con concurrencia limitada (4). Cada envío lleva una `ref` única (GUID) para correlacionar el callback.
7. **Cortes**: la API key rechazada (401/403) corta de inmediato; `FALLAS_SEGUIDAS_PARA_CORTAR` (5) fallas seguidas también.

## Alertas (máximo una por tipo y corrida)

| Tipo | Cuándo |
| ---- | ------ |
| `WsInterNoDisponible` | `/health` falla, o todas las solicitudes de la corrida fallaron |
| `CredencialInvalida` | ws-inter responde 401/403 |
| `CorridaCortada` | 5 fallas seguidas |
| `SinRespuesta` | Guías con 3 envíos sin callback |
| `ErrorNoControlado` | Excepción (p. ej. Mongo caído); además se relanza para que cuente como error de la lambda |

## Límite de tiempo

`Function.cs` corta la corrida a los 150 s (o 10 s antes del timeout de la lambda). Lo pendiente se retoma en la
siguiente; `interrumpida=true` en el resumen.

## Pruebas

`test/ApiLambdaNovedadesInter.Tests/Aplicacion/SolicitarNovedadesUseCaseTests.cs` y
`Dominio/ReglasTests.cs` (tabla completa del planificador).
