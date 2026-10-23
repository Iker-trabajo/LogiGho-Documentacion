---
autor: Iker Acevedo
fecha_creacion: 2026-10-22
estado: aceptada
---

# ADR-003 — Idempotencia y cero guías duplicadas

**Fecha:** 2026-10-22
**Estado:** Aceptada

---

## Contexto

En las lambdas actuales puede crearse la misma guía dos veces:

- `Generico.consumoGenerico` reintenta hasta 4 veces cualquier POST, incluida la creación en la transportadora (`ApiLambdaCrearGuia/Infraestructura/Servicios/Generico.cs:152-237`). Si la transportadora creó la guía pero la respuesta no llegó, el reintento crea otra.
- El estado `Cargado` se escribe **después** de crear la guía (`ApiLambdaCargarGuias/.../CargaInterUseCase.cs:121,317`).
- `PedidosInter` no tiene índice único por número de guía.
- Si el insert en `PedidosInter` falla, la guía queda creada en la transportadora y el pedido huérfano, sin aviso.

## Decisión

1. Header **`Idempotency-Key` obligatorio**. Índice único `{ClienteId, IdempotencyKey}` en `ApiSolicitudesGuia`, con hash del body.
   - Misma key y mismo body: se devuelve el resultado guardado.
   - Misma key y otro body: `422 IDEMPOTENCY_CONFLICTO`.
   - Misma key en curso: `409`.
2. **Estado persistido antes de llamar a la transportadora** (`CREANDO`), con máquina de estados: `RECIBIDA → RESERVADA → CREANDO → CREADA | FALLIDA | INCIERTA → ANULADA`.
3. **Nunca reintentar a ciegas el POST de creación.** Solo se reintenta si el error ocurrió antes de enviar (conexión rechazada, DNS).
4. Timeout o 5xx ambiguo → `INCIERTA`. El **reconciliador** consulta a la transportadora o alerta a operación; no se crea otra guía para esa solicitud hasta resolverla.
5. Los pasos posteriores (insert `PedidosInter`, etiqueta, inventario) son reintentables porque son idempotentes por número de guía; si fallan, la solicitud queda marcada y se reintenta desde la cola, no se pierde.
6. Mensajes SQS idempotentes por id de solicitud.

## Consecuencias

- El integrador debe generar una key única por guía (ej. su id de pedido).
- Alarma si hay solicitudes `INCIERTA` por más de 10 minutos.
