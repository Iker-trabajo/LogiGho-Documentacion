## Autor: Yadilton Torres

Fecha creacion: 2026-10-30

Estado: desarrollo

## Helper: novedades.mapper.ts

**Ubicación:** `helpers/novedades.mapper.ts`
**Tests:** `novedades.mapper.spec.ts`

---

## ¿Qué hace?

Convierte lo que devuelve el API (ya proyectado por los pipelines del repositorio) a `NovedadFila`. Los nombres de entrada son los **alias del `$project`**, no los campos crudos de Mongo.

| Función | Entrada | Qué hace |
| ------- | ------- | -------- |
| `filaDesdeSeguimientoInter(doc)` | `SeguimientoNovedadesInter` | Fila liviana de Inter: motivo, prioridad, consultas, ciudad, ubicación del último intento y el historial con la ubicación de cada intento |
| `aplicarDetalleInter(fila, doc, urlGuia)` | Detalle de Inter | Completa la fila (sin mutarla): trazabilidad ordenada por `orden`, destinatario, tipo de entrega, PDF y evidencias (`gestion_app`) ordenadas por `orden` |
| `filaDesdePedido(doc)` | `PedidosInter` | Fila completa de Envía o Servientrega; devuelve `null` si la transportadora no es una de las que muestra la vista |
| `enteroONull(v)` | — | Número entero ≥ 0 o `null` (algunos pedidos guardan `IntentosEntrega` como texto) |

---

## Envía vs. Servientrega en `filaDesdePedido`

```
Envía        Estado "EN NOVEDAD POR MEDELLIN"  → estadoInter "EN NOVEDAD", ciudad "MEDELLIN", etapa "novedad"
Servientrega Descripcion "Devolucion: SE NEGO A RECIBIR" → estadoInter "EN DEVOLUCIÓN", etapa "devolucion"
```

El `Estado` de Servientrega es interno ("EN PROCESAMIENTO"), así que no se muestra: se muestra la etapa, que es lo que se entiende. Un pedido de Servientrega **sin** mensaje de devolución queda como novedad normal.

---

## Detalles que cuidar

- `TipoEntrega` de `PedidosInter` llega como código: `1` → `ENTREGA EN DIRECCION`, `2` → `RECLAME EN OFICINA`.
- Las evidencias sin `s3_key` se descartan.
- Las ciudades de Inter vienen como `BUENAVISTA BOYACA\BOYA\COL`; `limpiarCiudad` deja solo el nombre.
