---
autor: Iker Acevedo
fecha_creacion: 2026-10-25
ultima_actualizacion: 2026-10-25
estado: vigente
---

# Runbook · alarma `ApiGuias-<entorno>-documento-ausente` (P-07)

## Qué significa

El worker intentó actualizar un documento de `CargaPedido` o `PedidosInter` **que él mismo había insertado** y ya no existe (borrado a mano, o una carrera con otro proceso). El worker **no reintenta a ciegas**: el documento no va a aparecer solo y la guía ya pudo crearse en la transportadora. Hizo cuatro cosas:

1. Publicó la métrica `DocumentoAusente` (dimensiones `Coleccion` y `Transportadora`).
2. Marcó la solicitud con `MotivoRevisionManual` en `ApiSolicitudesGuia` (dice qué documento y en qué paso).
3. **Conservó la reserva de cupo** (la guía puede existir en la transportadora y costarle dinero a la tienda).
4. Confirmó el mensaje: no va a la DLQ ni la reencola el reconciliador.

Un solo caso ya importa: hay cupo y, posiblemente, una guía real **sin pedido**.

## Qué hacer

1. **Abrir la solicitud.** En `ApiSolicitudesGuia`, buscar por el `solicitudId` del registro (`MotivoRevisionManual` no nulo). Mirar `Estado`, `NumeroGuia`, `PasosCompletados`, `IdCargaPedido`, `IdPedidoInter` y `Reserva`.
2. **Comprobar si el documento existe con otro `_id`.** Buscar en la colección que indica el motivo por `Numeropreenvio` / `NumeroPreenvio` igual al `NumeroGuia` (borrado manual y reinserción, o un proceso de depuración).
3. **Si la guía ya existe en la transportadora (`NumeroGuia` presente): no volver a crearla.** Reconstruir el documento que falta a partir de la solicitud (los campos salen de `Datos` y `Contexto`; el formato es el de `DocumentosPedido`) e insertarlo con el `_id` guardado en la solicitud. Después limpiar `MotivoRevisionManual` y poner `ConPasosPendientes: true` para que el reconciliador complete los pasos que faltan.
4. **Si no hay guía (`Estado` `Reservada` o `Fallida` sin número):** decidir con Iker entre marcarla `Fallida` (libera el cupo) o dejarla para reintento.
5. **Reencolar** desde la DLQ solo después de arreglar la causa.
6. **Anotar** la causa en la bitácora.

## Qué no hacer

- No borrar la solicitud: perdería la idempotencia y el rastro del cupo.
- No quitar `MotivoRevisionManual` sin haber arreglado el documento: el worker volvería a fallar igual.
- No liberar la reserva a mano si la guía existe en la transportadora.

## Alarmas parecidas

| Alarma | Significa |
| --- | --- |
| `ApiGuias-<entorno>-fallida-por-reintentos` (P-06) | Una solicitud agotó sus 5 intentos porque la transportadora no estuvo disponible: quedó `Fallida` y liberó su cupo. No hay que recuperar nada; avisar al integrador y revisar la transportadora |
| `ApiGuias-<entorno>-dlq` | Un mensaje agotó sus entregas por otro error |
