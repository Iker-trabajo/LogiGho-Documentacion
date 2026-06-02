## Autor: Iker Acevedo
Fecha creacion: 2026-06-02

Estado: produccion

# Lambda: ApiLambdaSincronizacionUnion

**Namespace:** `ApiLambdaSincronizacionUnion`

**Accionador:** Step Function

**AOT:** No

---

## ¿Qué hace?

Une los pedidos del último mes de PedidosInter con sus liquidaciones correspondientes LiquidacionesLogigho usando el número de guía como llave, y sincroniza el resultado en la colección UnionPedidosLiquidaciones. Por cada pedido genera un documento por cada liquidación asociada, o un documento solo con datos del pedido si no tiene liquidaciones. En caso de tener mas de una liquidación genera un documento por cada liquidacion, con el pedido correspondiente.

---

## Accionador

| Tipo          | Descripción                                                  |
| ------------- | ------------------------------------------------------------ |
| `Step Function` | Invocación periódica automática. No recibe parámetros. |

---

## Request

No aplica. La lambda no recibe parámetros. Siempre procesa el último mes completo.

---

## Response

### Exitoso

```json
{
  "error": false,
  "mensaje": "Sincronizacion completada exitosamente",
  "totalRegistros": 140657,
  "resultado": {
    "totalPedidosProcesados": 80010,
    "totalLiquidacionesEncontradas": 114965,
    "totalDocumentosUnion": 140657,
    "fechaSincronizacion": "2026-06-02 13:56:47"
  }
}
```

### Errores

| Código | Cuándo         |
| ------ | -------------- |
| `500`  | Error interno  |

---

## Flujo interno

```
FunctionHandler
  -> ObtenerLiquidacionesLogighoAsync
     -> MongoDB: LiquidacionesLogigho (lectura último mes)
        -> Construye índice en memoria (guía normalizada -> liquidaciones)
        -> Libera lista original de liquidaciones (GC.Collect)

  -> ObtenerNumeropreenviosUltimoMesAsync
     -> MongoDB: PedidosInter (solo campo Numeropreenvio)

  -> EliminarPorNumeropreenviosAsync
     -> MongoDB: UnionPedidosLiquidaciones (delete por Numeropreenvio)

  -> ProcesarPedidosEnLotesAsync (cursor lotes de 500)
     -> MongoDB: PedidosInter (lectura último mes)
        -> Por cada pedido busca liquidaciones en índice
        -> Construye documentos de unión
        -> InsertarDocumentosAsync
           -> MongoDB: UnionPedidosLiquidaciones (insert por lote)
```

---

## Dependencias externas

| Servicio               | Uso                                               |
| ---------------------- | ------------------------------------------------- |
| `Seguridad.Encripcion` | Descifra la cadena de conexión MongoDB (AES256ECB) |

---

## Historial de cambios

| Fecha      | Autor        | Cambio                                                                                                      |
| ---------- | ------------ | ----------------------------------------------------------------------------------------------------------- |
| 2026-06-02 | Iker Acevedo | Refactor de memoria: cursor por lotes de 500, liberación explícita de liquidaciones, separación de delete e insert |

---

## Observaciones

> Deuda técnica, comportamientos especiales, decisiones no obvias del código.

- `SincronizarUnionPedidosLiquidacionesAsync` quedó en el repositorio sin uso. Pendiente eliminar una vez confirmado el correcto funcionamiento en producción.
- El filtro de datos usa el `_id` de MongoDB (timestamp de inserción) y no campos de fecha del negocio como `FechaCarga`.
- `GC.Collect()` se llama manualmente después de liberar las liquidaciones. Justificado por el volumen de datos y memoria limitada de Lambda.
- Se crea el índice temporal `idx_Numeropreenvio_temp` antes del delete y se elimina en el `finally`. Si la lambda falla antes del `finally`, el índice puede quedar huérfano — se maneja con el catch de `IndexOptionsConflict` en la siguiente ejecución.
- RAM estable en ~1.5 GB independiente del volumen de datos. Solo el tiempo crece linealmente con más pedidos.
