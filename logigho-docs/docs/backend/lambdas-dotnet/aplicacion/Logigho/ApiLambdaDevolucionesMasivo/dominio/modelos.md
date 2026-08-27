## Autor: Iker Acevedo

Fecha creacion: 2026-08-26

Estado: produccion

## Dominio: Modelos y Enums

**Ubicación:** `Dominio/Modelos/`, `Dominio/Enums/`, `Dominio/Servicios/`

---

## `JobDevolucion` — el documento que orquesta todo

Colección `JobsDevolucion`. Es el único canal de comunicación entre Iniciador, Worker y front — ninguno llama a otro directamente. `JobId` se mapea como el `_id` del documento (`JobRepository.RegistrarMapeo`), así que la unicidad la garantiza Mongo, no código.

| Campo | Tipo | Notas |
| ----- | ---- | ----- |
| `JobId` | `string` | `_id`. `Guid.NewGuid("N")`, no cambia entre continuaciones |
| `Usuario` | `string` | Un job pertenece a un usuario, no a una tienda — un archivo puede traer guías de varias tiendas |
| `Estado` | `EstadoJob` | `Pendiente` → `EnProceso` → `Completado` / `Fallido` |
| `GuiasPendientes` | `List<string>` | La cola real. Es lo que hace posible la continuación sin reprocesar |
| `Rechazadas` | `List<GuiaProcesada>` | **Único** array que se acumula completo. Ver [ADR-002](../adr/ADR-002-jobdevolucion-solo-contadores.md) |
| `GuiasTotal` | `int` | Aceptadas al crear el job |
| `TotalResueltas`, `TotalYaProcesadas`, `TotalRechazadas` | `int` | Contadores, se incrementan con `$inc` |
| `GuiasProcesadas` | `int` (calculado) | Suma de los 3 contadores |
| `Continuaciones` | `int` | Cuántas veces se reinvocó el Worker por falta de tiempo |
| `PuedeContinuar` | `bool` (calculado) | `Continuaciones < MaximoContinuaciones (3)` |
| `MensajeError` | `string?` | Solo si `Estado == Fallido` |
| `FechaInicio` | `DateTime` | `UtcNow`, UTC real |
| `FechaUltimaActividad` | `DateTime` | Se refresca en cada lote — es lo que detecta un Worker caído |
| `FechaFin` | `DateTime?` | Solo al completar o fallar |

```csharp
public const int MaximoGuiasPorJob = 3000;
public const int MaximoContinuaciones = 3;
```

---

## `GuiaProcesada` — el resultado de resolver una guía individual

Guarda todo lo necesario para escribir en `InventarioDevolucion` sin volver a consultar el pedido.

| Campo | Tipo | Notas |
| ----- | ---- | ----- |
| `Guia` | `string` | Tal cual la subió el operario |
| `Estado` | `EstadoResolucion` | `YaProcesada` / `ResueltaDirecta` / `ResueltaInter` / `Rechazada` |
| `Motivo` | `MotivoRechazo?` | Solo si `Rechazada` |
| `PedidoId`, `Numeropreenvio`, `GuiaDevolucion`, `Transportadora`, `Tienda`, `Observaciones`, `CantidadTotalProductos` | — | Nulos si no se encontró pedido |
| `GuiaOriginalExtraida` | `string?` | La que reportó la transportadora, se haya encontrado el pedido o no — es lo que permite auditar huérfanas |
| `FechaDevolucionTransportadora` | `string?` | Normalizada al formato de `PedidosInter."Fecha Devolucion"` |
| `Fecha` | `DateTime` | `UtcNow` — UTC real (ver [ADR-004](../adr/ADR-004-fechas-utc.md)) |

**Propiedades calculadas (no se persisten — `JobRepository.RegistrarMapeo` las `UnmapMember`):**

```csharp
FueResuelta                       => Estado != Rechazada
EsRecienResuelta                  => Estado is ResueltaDirecta or ResueltaInter
RequiereActualizarInventario      => EsRecienResuelta
MereceReintentoConOtraTransportadora => Estado == Rechazada && Motivo == SinGuiaOriginalEnRespuesta
```

`EsRecienResuelta` es distinto de `FueResuelta`: `FueResuelta` también incluye `YaProcesada` (se resolvió en una corrida **anterior**), y esas no deben volver a marcarse como devolución completada ni mover inventario otra vez.

---

## Enums

### `EstadoJob`
`Pendiente` → `EnProceso` → `Completado` | `Fallido`

### `EstadoResolucion`
`YaProcesada` | `ResueltaDirecta` | `ResueltaInter` | `Rechazada`

`ResueltaDirecta` = match directo en `PedidosInter` (fase A). `ResueltaInter` = resuelta vía consulta a una transportadora (fase B/C) — el nombre es histórico, aplica igual a Envía.

### `MotivoRechazo`

| Motivo | Cuándo |
| ------ | ------ |
| `GuiaNoExisteEnSistema` | Sin match por ningún campo, y la transportadora tampoco devolvió una guía original |
| `OriginalNoExisteEnSistema` | La transportadora SÍ devolvió una guía original, pero no existe en `PedidosInter` — guía **huérfana**, se guarda la extraída para reclamar |
| `SinGuiaOriginalEnRespuesta` | La transportadora respondió pero no había ningún número extraíble en el texto — candidata a reintento con otra transportadora |
| `PedidoAnulado` | El pedido existe pero está anulado |
| `SinPermisoTienda` | El pedido es de una tienda que el usuario no tiene asignada |
| `ErrorConsultaExterna` | Falla técnica consultando la API externa — no es culpa de la guía |

---

## `ContextoUsuario`

```csharp
Email: string
TiendasAsignadas: IReadOnlyList<string>
TieneAccesoATodas => TiendasAsignadas.Any(t => t == "Todas")
PuedeOperarTienda(tienda) => TieneAccesoATodas || TiendasAsignadas.Contains(tienda)
```

Con `"Todas"` en la lista, **ninguna** tienda se rechaza jamás — es el comodín de administrador. El nombre de la tienda del pedido no importa, solo si está en la lista (o si hay comodín).

---

## `PedidoEncontrado`

Proyección mínima de `PedidosInter` que necesita el flujo — nunca se trae el documento completo, solo `_id`, `Numeropreenvio`, `GuiaDevolucion`, `Estado`, `Tienda`, `Transportadora`, `Observaciones`, `CantidadTotalProductos`.

---

## `NormalizadorGuia` (Dominio/Servicios)

Reglas de formato de guías puras, sin dependencias — viven en el dominio porque son la causa más común cuando el sistema dice "la guía no se encuentra".

```csharp
Limpiar(guia)          quita todo lo que no sea digito
TryComoNumero(guia)    intenta parsear como long, para el filtro dual texto/numero de Mongo
ConFormatoEnvia(guia)  rellena con ceros a la izquierda hasta 12 digitos (lo que espera la API de Envia)
PareceInter(guia)      13 digitos, empieza en "30"
PareceEnvia(guia)      12 digitos
```
