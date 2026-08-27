## Autor: Iker Acevedo

Fecha creacion: 2026-08-26

Estado: produccion

## Infraestructura: Estrategias y clientes de transportadora

**Ubicación:** `Infraestructura/Estrategias/`, `Infraestructura/Servicios/`

---

## Arquitectura: Strategy + Factory, sin switch por transportadora

```
IEstrategiaDevolucion (interfaz)
  Transportadora: string
  PuedeResolver(guia): bool
  ResolverLoteAsync(guias): ResultadoResolucion

EstrategiaConsultaExterna implements IEstrategiaDevolucion
  — UNA sola clase para cualquier transportadora que exponga consulta por guia.
  — recibe un IClienteTransportadora inyectado (ClienteInter o ClienteEnvia)
  — la orquestacion (preguntar -> extraer original -> buscar en PedidosInter -> armar resultado)
    es identica sin importar la transportadora

FactoryEstrategiaDevolucion
  — reparte el lote preguntandole a cada estrategia si "PuedeResolver" el formato
  — sin switch: agregar una transportadora es registrar una clase mas (Open/Closed)
  — EstrategiaSinResolucion es el respaldo: nunca se ofrece voluntaria, solo la
    usa el factory cuando NINGUNA estrategia reconocio el formato
```

Registrar una transportadora nueva mañana: implementar `IClienteTransportadora`, agregar una línea al arreglo `estrategias` en `WorkerHandler.Construir()`. Ni `EstrategiaConsultaExterna`, ni `FactoryEstrategiaDevolucion`, ni `ProcesarJobUseCase` cambian.

---

## `EstrategiaConsultaExterna` — la orquestación compartida

```
ResolverLoteAsync(guias)
  1. cliente.ConsultarLoteAsync(guias)          UNA tanda contra la API de la transportadora
  2. originales = consultas con GuiaOriginal, sin repetir
  3. repositorio.BuscarPorNumeropreenvioAsync(originales)   UNA consulta a Mongo, no una por guia
  4. Interpretar(consulta, pedidos) por cada una:

       consulta.Fallo?                    -> Rechazada, ErrorConsultaExterna
       !consulta.TieneGuiaOriginal?        -> Rechazada, SinGuiaOriginalEnRespuesta
       pedidos no tiene GuiaOriginal?       -> Rechazada, OriginalNoExisteEnSistema  (HUERFANA)
       si no                                -> ResueltaInter, con todos los datos del pedido
```

La guía huérfana (`OriginalNoExisteEnSistema`) se logea con `Log.Warn(_ctx, "guia-huerfana", ...)` y conserva `GuiaOriginalExtraida` — es el número que hay que reclamarle a la transportadora, porque nos devolvió un paquete que nunca despachamos.

---

## `ClienteInter` — Interrapidísimo

Necesita **dos llamadas por guía**, y el orden importa:

```
1. Rastreo(guia de devolucion)   -> "DiceContener" trae el numero original en texto libre
2. Estados(guia ORIGINAL)         -> la fecha del estado de devolucion
```

El paso 2 se consulta **sobre la guía original**, no sobre la escaneada — verificado contra datos reales, la guía de devolución nunca tiene ningún estado con "Devoluci" (Inter la marca "Entregada", porque efectivamente se entregó en nuestra bodega). El estado de devolución vive en la guía inicial, como `"Devolución ratificada"`.

```
"Devolución ratificada"  cuenta como devolucion
"Devolucion Regional"    NO cuenta — es una devolucion entre sedes de Inter,
                          no significa que la mercancia volvio a nuestra bodega
```

**Formato de guía Inter:** 13 dígitos, empieza en `"30"`.

**Token cacheado por contenedor** (`_token` estático, `_tokenVence`): el módulo legacy pedía token en **cada** invocación — con 1.000 guías eran 1.000 autenticaciones innecesarias. Acá se pide una vez, vence a los 20 minutos, y un `SemaphoreSlim` evita que 10 tareas concurrentes pidan 10 tokens a la vez en el primer uso tras un cold start (double-check locking).

**Concurrencia:** `CONCURRENCIA_INTER` (por defecto 10) — el rastreo es una llamada por guía; los estados de las originales sí aceptan varias por llamada, en lotes de 15 (`TamanoLoteMaximo`).

---

## `ClienteEnvia`

Su API expone la guía en la **URL** (`GET .../ConsultaGuia/{guia}`), no admite lote — una llamada por guía, compensado con concurrencia controlada (`CONCURRENCIA_ENVIA`, por defecto 5).

**Formato de guía Envía:** 12 dígitos (`ConFormatoEnvia` rellena con ceros a la izquierda si hace falta).

**La guía original viene embebida en texto libre**, campo `anotaciones`:

```csharp
private static readonly Regex GuiaOriginal = new(@"\b\d{12}\b", RegexOptions.Compiled);
```

Un único número de 12 dígitos por anotación, verificado sobre 124 muestras reales. El `\b` (límite de palabra) evita capturar los dígitos de un número más largo por accidente.

Si no hay match, se devuelve `ConsultaGuiaResultado.SinDevolucion(guia)` — **no** es un fallo técnico (`Fallo=false`), es la transportadora respondiendo correctamente pero sin dato útil, y eso es lo que en `EstrategiaConsultaExterna.Interpretar` cae en `SinGuiaOriginalEnRespuesta` (reintentable con otra transportadora), no en `ErrorConsultaExterna`.

**Fecha normalizada al mismo formato que Inter**, para que `"Fecha Devolucion"` quede homogéneo en `PedidosInter` sin importar la transportadora — Envía entrega fecha (`dd/MM/yyyy`) y hora por separado, se combinan y parsean a `yyyy-MM-ddTHH:mm:ss.fff`.

---

## Tabla comparativa

| | ClienteInter | ClienteEnvia |
| - | ------------ | ------------ |
| Formato de guía | 13 dígitos, empieza en `30` | 12 dígitos |
| Llamadas por guía | 2 (rastreo + estados) | 1 |
| Soporta lote | Estados sí (15), rastreo no | No |
| Concurrencia por defecto | 10 | 5 |
| Origen de la guía original | Regex sobre `DiceContener` | Regex sobre `anotaciones` |
| Autenticación | Token temporal cacheado 20 min | Ninguna (URL pública con la guía) |

---

## `ServicioInventario`

`POST /actualizaInventarioLotes` — **un solo POST con todo el lote**, en vez de una llamada por guía. El trabajo pesado del lado de esa lambda es fijo por petición, no proporcional al lote: agrupar más guías no lo encarece.

`MaximoGuiasPorPeticion = 500` — se parte en trozos para no acercarse al límite de 15 minutos de esa lambda.

`INVENTARIO_SIMULADO=true` registra en log lo que habría enviado sin llamar al servicio real — la única forma segura de probar el flujo completo sin mover inventario real.

Un fallo acá **no tumba el job** — la devolución ya quedó registrada en `InventarioDevolucion` y el pedido sigue siendo elegible en una corrida posterior de `ApiLambdaActualizacionInventario`, que filtra por su propia marca de procesado.
