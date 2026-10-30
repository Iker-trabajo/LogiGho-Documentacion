## Autor: Yadilton Torres

Fecha creacion: 2026-10-30

Estado: desarrollo

## Servicio: NovedadesRepository

**Ubicación:** `helpers/novedades.repository.ts`

---

## ¿Qué hace?

Único punto del módulo que habla HTTP. Ningún componente ni el store importa `ConsumoGenericoService`, `DecompressionService` ni `GetObjectService` directamente. Todo es **de solo lectura**: pipelines de agregación sobre `metodoGenerico` (mismo patrón base64 que `relacion-despacho`) y `getObject` para las fotos.

| Método | Fuente | Uso |
| ------ | ------ | --- |
| `listarInter(cerradasDesde)` | `SeguimientoNovedadesInter` | Activas (siempre) y cerradas desde una fecha (solo al abrir la pestaña) |
| `listarPedidos()` | `PedidosInter` | Envía en novedad + Servientrega en devolución, en una sola consulta |
| `detalleInter(fila)` | `SeguimientoNovedadesInter` + `PedidosInter` | Trazabilidad, destinatario, tipo de entrega, `gestion_app` y PDF de una guía |
| `fotoEvidencia(s3Key)` | S3 `wsinter-evidencias-prod` vía `getObject` | Foto de la novedad como `data:image/...;base64` |

---

## Por qué dos consultas y no una

La base es **DocumentDB**: no soporta `$facet`, y la Lambda `ApiLambdaCrudGenericoAOT` bloquea `$lookup` y `$unionWith`. No se puede unir `SeguimientoNovedadesInter` con `PedidosInter` en el servidor, así que se piden por separado y se juntan en el store.

---

## `listarPedidos` — cada transportadora con su regla

```typescript
$or: [
  { Transportadora: 'ENVIA', Estado: { $regex: '^EN NOVEDAD' } },
  { Transportadora: 'SERVIENTREGA',
    Estado: { $in: ESTADOS_DEVOLUCION_SERVIENTREGA },          // ['EN PROCESAMIENTO', 'Devolucion']
    'Descripcion Asociada': { $regex: '^Devoluci' } },
]
```

- **Regex anclado y sin la opción `i`:** así Mongo recorre solo el tramo `EN NOVEDAD…` del índice `Transportadora_1_Estado_1`. Con `i` (o sin `^`) leería todos los pedidos de la transportadora.
- **Sin `$sort`:** ordenar por otro campo obligaría a leer todo antes del `$limit`; el orden lo pone el store.
- **Servientrega:** su `Estado` nunca dice novedad. Las que vienen de regreso tienen un estado abierto y el mensaje `Devolucion: <motivo>`. Los estados cerrados ("Devolucion Completada", "Pagada", "Anulada") no entran.

---

## `completarIntentosInter` — dato cruzado

Los intentos de entrega de Inter no están en `SeguimientoNovedadesInter` sino en `PedidosInter.IntentosEntrega`. Se cruzan por número de guía (índice `Numeropreenvio_1`) en tandas, para no pasar el largo de la URL. Es un extra: si una tanda falla, esas guías quedan sin intentos y la vista sigue.

---

## Proyección liviana en el listado

El listado de Inter **no** trae `UltimaRespuesta` completa: es pesada y tiene datos personales (remitente, destinatario, liquidación). Solo trae lo que pinta la tabla. `detalleInter` pide lo demás al abrir una guía:

```typescript
{ Flujo: '$UltimaRespuesta.flujo', Destinatario: '$UltimaRespuesta.destinatario',
  TipoEntrega: '$UltimaRespuesta.envio.tipo_entrega', GestionApp: '$UltimaRespuesta.gestion_app' }
```

El PDF de la guía sale de `PedidosInter.UrlGuia`. Si esa consulta falla, el detalle se muestra igual sin PDF.

---

## Filtro por tienda

`tiendasUsuario()` lee `sessionStorage.getItem('tiendas_asignadas')`. Si el usuario tiene `"Todas"`, no se filtra; si no, cada consulta agrega `Tienda: { $in: [...] }`.

---

## Constantes

| Constante | Valor | Para qué |
| --------- | ----- | -------- |
| `LIMITE_CONSULTA` | 5000 | Tope de seguridad por consulta; si se alcanza, el store avisa carga parcial |
| `ESTADOS_DEVOLUCION_SERVIENTREGA` | `['EN PROCESAMIENTO', 'Devolucion']` | Estados abiertos de Servientrega mientras el envío regresa |
| `BUCKET_EVIDENCIAS` | `wsinter-evidencias-prod` | Bucket de las fotos de la novedad de Inter |

---

## Observaciones

- `fotoEvidencia` usa `getObject(bucket, s3Key, 'false', 'true')` y devuelve el base64 listo para un `<img>`. Si la Lambda de `getObject` no tiene permiso sobre el bucket, la foto queda en error y el motivo se ve en la consola (`[novedades] no se pudo cargar la foto de evidencia`).
- Índices que usa: `PedidosInter.Transportadora_1_Estado_1`, `PedidosInter.Numeropreenvio_1`, `SeguimientoNovedadesInter.ix_estado_seguimiento`.
