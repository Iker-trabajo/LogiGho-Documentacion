---

## Autor:
Fecha creacion: 2026-08-20
Estado: produccion

# Modelo: cambio-entrega-inter.models.ts

**Ubicación:** `src/app/views/logistica/cambio-entrega-inter/models/cambio-entrega-inter.models.ts`

---

## `CambioEntregaInter`

Contrato del documento de la colección `AuditoriaCambioEntregaInter` (ver [lambda backend](../../../../../backend/lambdas-dotnet/aplicacion/interrapidismo/ApiLambaProcesoAutomaticoInter/ApiLambaProcesoAutomaticoInter.md)).

| Campo | Tipo | Descripción |
| ----- | ---- | ----------- |
| `_id`, `NumeroGuia`, `PedidoId` | `string` | Identificadores — nunca numéricos, aunque `NumeroGuia` "parezca" un número |
| `Direccion`, `Ciudad`, `Departamento`, `Destinatario`, `Telefono` | `string` | Denormalizados al momento de la primera detección |
| `IdTienda`, `Tienda`, `Asesor` | `string` | Denormalizados |
| `TotalRecaudo` | `number` | Cantidad — se suma/promedia en KPIs y gráficas |
| `FechaCreacionPedido` | `string` | Cuándo se creó el PEDIDO, antes de que Inter lo tocara — **no confundir** con `FechaPrimeraDeteccion` |
| `Estado` | `string` | Congelado en `"Reclame en oficina"` — es el registro de la detección, no el estado actual |
| `DescripcionAsociada` | `string` | Descripción que Inter reportó junto al evento |
| `TipoEntrega` | `number` | Flag `1`/`2` — no es un identificador |
| `FechaEstadoOficina` | `string` | Timestamp que **Inter** reportó para el evento (no el nuestro) |
| `FechaPrimeraDeteccion` | `string` | Cuándo **nuestro sistema** detectó el cambio por primera vez |
| `FechaUltimaDeteccion` | `string` | Última corrida en la que la guía seguía en oficina |
| `EstadoGestion` | `'Pendiente' \| 'Gestionada'` | Lo controla el usuario desde el front |
| `GestionadoPor?` | `GestionadoPor` | Rastro del último cambio de gestión — solo lo escribe el front |
| `EstadoFinal?` | `'Entrega' \| 'Devolucion' \| null` | Desenlace final ante Inter. Ausente = sigue en oficina |
| `FechaResolucion?` | `string` | Cuándo se resolvió (junto con `EstadoFinal`) |

## `GestionadoPor`

| Campo | Tipo | Descripción |
| ----- | ---- | ----------- |
| `Usuario` | `string` | Quién hizo el último cambio |
| `FechaModificacion` | `string` | ISO con offset `-05:00` explícito (hora Colombia), no UTC |
| `Estado` | `EstadoGestion` | Estado resultante después del cambio (no el anterior) |

## `CambioEntregaFiltros`

Dos rangos de fecha **independientes** — filtran por campos distintos, no confundirlos:

| Campo | Filtra sobre |
| ----- | ------------- |
| `fechaDesde` / `fechaHasta` | `FechaPrimeraDeteccion` |
| `fechaCreacionDesde` / `fechaCreacionHasta` | `FechaCreacionPedido` |
| `busqueda`, `estadoGestion`, `tiendasSeleccionadas` | resto de filtros de la tabla |

## `CambioEntregaKpis`

`{ total, pendientes, gestionadas }` — derivados de `EstadoGestion`, calculados en el componente raíz.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-31 | Iker Acevedo | Modelo inicial: `CambioEntregaInter`, `CambioEntregaFiltros`, `GestionadoPor`. |
| 2026-08-18 | Iker Acevedo | `EstadoFinal`/`FechaResolucion`. Segundo rango de fecha (`fechaCreacionDesde`/`fechaCreacionHasta`). |
| 2026-08-20 | Iker Acevedo | `TotalRecaudo`/`TipoEntrega` de `string` a `number` — ver [ADR-001](../adr/ADR-001-desenlace-final.md). |

---

## Observaciones

- `CambioEntregaDesenlace` (tipo intermedio para el resumen de la dona) se eliminó del modelo — el modal de gráficas (`cambio-entrega-analisis`) calcula ese resumen internamente a partir de `items: CambioEntregaInter[]`, no necesita un tipo compartido.
