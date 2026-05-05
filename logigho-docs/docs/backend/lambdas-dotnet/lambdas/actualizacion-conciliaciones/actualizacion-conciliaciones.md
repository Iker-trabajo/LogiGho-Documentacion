---

## Autor: Iker Acevedo

Fecha creacion: 2026-05-05  
Ultima actualizacion: 2026-05-05  
Estado: produccion

# Lambda: ApiLambdaActualizacionConciliaciones

**Namespace:** `ApiLambdaActualizacionConciliaciones`  
**Trigger:** API Gateway  
**AOT:** Sí

---

## ¿Qué hace?

Traelos pedidos de los últimos 6 meses con los registros de fecha de entrega o devolucion de todas las transportadoras integradas (Interrapidísimo, Envia, X-Cargo, Servientrega, D2E) y actualiza el campo `Estado` a `"Pagada"` en `PedidosInter` cuando encuentra coincidencia por número de guía. Al actualizar agrega 2 nuevos campos sobre el pedido con la misma guia: Pago y FechaPagocl

---

## Endpoint


| Método | Ruta                         | Auth          |
| ------ | ---------------------------- | ------------- |
| `POST` | `/conciliaciones/actualizar` | Token Cognito |


---

## Request

```json
{
  "TokenCognito": "string"
}
```


| Campo          | Tipo     | Requerido | Descripción                                                       |
| -------------- | -------- | --------- | ----------------------------------------------------------------- |
| `TokenCognito` | `string` | No*       | Token Cognito del usuario. También se acepta en el header `Token` |


>  Al menos uno de los dos (header `Token` o body `TokenCognito`) debe estar presente.

---

## Response

### Exitoso

```json
[
  {
    "TotalPedidosProcesados": 1000,
    "ActualizacionesExitosas": 850,
    "ActualizacionesFallidas": 150,
    "FechaProcesamiento": "2026-05-05 10:30:00"
  }
]
```

### Errores


| Código | Cuándo                                                                         |
| ------ | ------------------------------------------------------------------------------ |
| `500`  | Excepción no controlada — se lanza como `Exception` con el mensaje serializado |


---

## Flujo interno

```
FunctionHandler (Function.cs)
  → ProcesarConciliacionUseCase.ProcesarConciliacionesAsync()
      ├── IConciliacionRepository.ProcesarColeccionesConciliacionAsync()
      │     MongoDB (paralelo): ConciliacionPagos · ConciliacionPagosXcargo
      │                         ConciliacionPagosEnvia · ConciliacionPagosServientrega
      ├── IConciliacionRepository.ObtenerPedidosInterUltimos6MesesAsync()
      │     MongoDB: PedidosInter (filtro: Estado ≠ "Pagada", con fecha entrega o devolución)
      ├── IConciliacionRepository.ObtenerColeccionAsync(strategy.NombreColeccion)
      │     MongoDB: ConciliacionPagosD2E  [por cada IConciliacionTransportadora registrada]
      │
      └── ProcesarLotePedidosSecuencialAsync()   [lotes de 500 pedidos]
            ├── Busca en índice Inter   → ActualizarPedidoConPago()
            ├── Busca en índice Envia   → ActualizarPedidoConPagoEnvia()
            ├── Busca en índice X-Cargo → ActualizarPedidoConPagoXcargo()
            ├── Busca en índice Servientrega → ActualizarPedidoConPagoServientrega()
            └── Busca en índices Strategy (D2E) → IConciliacionTransportadora.ActualizarPedidoAsync()
                  MongoDB: PedidosInter (UpdateOne por número de guía)
```

---

## Transportadoras y sus colecciones


| Transportadora  | Colección MongoDB               | Patrón       | Campo guía                                   | Campo fecha          | Campo valor        |
| --------------- | ------------------------------- | ------------ | -------------------------------------------- | -------------------- | ------------------ |
| Interrapidísimo | `ConciliacionPagos`             | Legacy       | `Guias` (puede ser lista separada por comas) | `Fecha`              | `Valor_total`      |
| Envia           | `ConciliacionPagosEnvia`        | Legacy       | `Guia`                                       | `Fec_Traslado`       | `Valor_Producto`   |
| X-Cargo         | `ConciliacionPagosXcargo`       | Legacy       | `Guia`                                       | `Fecha_consignacion` | `Monto_confirmado` |
| Servientrega    | `ConciliacionPagosServientrega` | Legacy       | `GUIA`                                       | `FECHA_FACTURACION`  | `VALOR_MOVILIZADO` |
| D2E             | `ConciliacionPagosD2E`          | **Strategy** | `NumeroGuia`                                 | `Fecha`              | `ValorRecaudar`    |


---

## Dependencias externas


| Servicio | Uso                                                                                   |
| -------- | ------------------------------------------------------------------------------------- |
| MongoDB  | Lectura de colecciones de conciliación y `PedidosInter`; escritura del estado de pago |


Variables de entorno requeridas en producción:


| Variable          | Descripción                                        |
| ----------------- | -------------------------------------------------- |
| `CADENA_CONEXION` | Cadena de conexión MongoDB cifrada con AES-256-ECB |
| `DATABASE_NAME`   | Nombre de la base de datos                         |


---

## Historial de cambios


| Fecha      | Autor        | Cambio                                                                                                                                                                                                                      |
| ---------- | ------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 2026-05-05 | Iker Acevedo | Integración de D2E mediante patrón Strategy. Nuevos: `IConciliacionTransportadora`, `ConciliacionD2EStrategy`, `ObtenerColeccionAsync`. El use case ahora itera sobre `_strategies` antes de finalizar el cruce por pedido. |


---

## Observaciones

- Las cuatro transportadoras legacy (Inter, Envia, X-Cargo, Servientrega) mantienen su lógica de cruce hardcodeada dentro de `ProcesarLotePedidosSecuencialAsync`. D2E fue la primera en incorporarse con Strategy; las legacy se migrarán gradualmente.
- `Numeropreenvio` en `PedidosInter` puede venir como `string`, `long`, `float` o BSON `$numberLong`. `ExtraerNumeroGuia()` normaliza todos los casos a string antes de la búsqueda en el índice.
- Los índices se construyen en memoria al inicio del proceso (antes del loop de lotes). El cruce posterior es O(1) por pedido.
- Los pedidos se procesan secuencialmente en lotes de 500 para evitar problemas de concurrencia en MongoDB.

