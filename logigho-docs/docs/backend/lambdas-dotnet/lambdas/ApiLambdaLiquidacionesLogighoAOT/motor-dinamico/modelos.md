## Autor: Iker Acevedo
Fecha creación: 2026-09-16  
Estado: preprod

# Modelos de dominio

Viven bajo `LiquidacionDinamica/Dominio/Modelos/`. Espejo 1:1 del front en [`liquidacion-transportadora.models.ts`](../../../../../frontend/views/director-de-operaciones/costos-transportadora/modelos/liquidacion-transportadora-models.md). **Cambio en backend = cambio en frontend también.**

---

## Configuración

### `ConfiguracionTransportadora`

Espejo de un documento de `ConfiguracionLiquidacionTransportadora`. Un documento por transportadora.

| Campo | Tipo | Descripción |
|---|---|---|
| `IdTransportadora` | `int` | ID de la colección `Transportadora` |
| `NombreEnPedidos` | `string` | Valor exacto (normalizado) como llega en `PedidosInter.Transportadora` — si no calza, el motor no puede asociar pedidos entrantes con esta configuración |
| `MotorLiquidacion` | enum | `Dinamica` \| `Legacy` \| `Sombra` \| `Deshabilitado` |
| `PoliticaPeso` | enum | `TablaIncrementosConTope` \| `PorcentajeSobreFleteTransportadora` — ver [Políticas de peso](politicas-peso.md) |
| `TopeKilosTabla` | `int?` | Requerido (≥2) si la política es tabla; debe ser `null` si es porcentaje |
| `AplicaMaximoServicioSeguro` | `bool` | Si es `true`, comisión de recaudo y seguro también respetan "nunca menos que lo cobrado por la transportadora" (hoy solo TCC) |
| `TrayectoPorDefecto` | `string?` | Código de trayecto a usar cuando el pedido trae uno no reconocido |

### `TrayectoTransportadora`

Lista blanca de códigos de ruta válidos por transportadora (`TrayectosTransportadora`). Solo catálogo — no lleva peso ni tarifa. `Codigo`, `NombreVisible`, `Orden`.

### `TarifaTransportadoraTienda`

Espejo de `TarifasTransportadoraTienda`: cuánto le cuesta liquidar una guía a una tienda puntual, o la Genérica de toda la transportadora.

| Campo | Descripción |
|---|---|
| `Alcance` | `Tienda` \| `Generico` |
| `IdTienda` | Requerido si `Alcance=Tienda`; `null` si `Generico` |
| `TipoLiquidacion` | `Entrega` \| `Devolucion` — cada combinación tienda+tipo tiene su propia tarifa |
| `PorcentajeRecaudo`, `PorcentajeSeguro` | % de comisión/seguro |
| `PorcentajeKiloAdicional` | % de recargo sobre el flete ya calculado, cuando el peso supera el tope de la tabla |
| `MontoMinimoServicio` | Piso fijo de servicio/seguro, sin importar el % configurado — la mayoría de transportadoras tiene uno |
| `Trayectos` | Lista de `TrayectoTarifa`, uno por cada trayecto que esta tarifa cubre |

### `TrayectoTarifa` / `IncrementoPeso`

`TrayectoTarifa`: flete base (1kg) de un trayecto puntual dentro de una tarifa, más su lista de `IncrementosPeso` (solo tiene efecto bajo la política de tabla). `IncrementoPeso`: `Kilos` + `IncrementoSobreKiloAnterior` — un escalón de la tabla.

### `CostosOperativosTienda`

Espejo de `AdministracionCostos` (por tienda, no por transportadora): porcentajes descuento flete/confirmación/adelanto (texto crudo tal cual Mongo) + montos fijos (`PicyPac`, `OtroPorcentajeVenta`, etc.). Parseados durante construcción liquidación.

---

## Pedido

### `PedidoLiquidable`

Subconjunto limpio (tipos normalizados, no `JToken` crudo) de documento `PedidosInter`. Lo mínimo para clasificar/validar/tarifar: Numeropreenvio, IdTienda, Transportadora, estado, fechas entrega/devolución, trayecto, peso.

⚠️ **No incluye:** dropshipper, marketing, referido (entran más adelante, resueltos por calculadoras propias).

---

## Resultados de cálculo

Cada uno es la salida de una Calculadora — ver [Calculadoras](calculadoras.md) para el detalle de cómo se llenan.

| Modelo | Salida de |
|---|---|
| `ResultadoBaseRecaudo` | `CalculadoraBaseRecaudo` |
| `ResultadoFleteTienda` | `CalculadoraFleteTienda` |
| `ResultadoCalculoEntrega` | `CalculadoraEntrega` |
| `ResultadoCalculoDevolucion` | `CalculadoraDevolucion` |
| `ResultadoCalculoDropshipper` | `CalculadoraDropshipper` |
| `ResultadoPagoAdicional` | `CalculadoraMarketing` / `CalculadoraReferido` (comparten el mismo modelo de salida) |
| `ContextoCalculoFlete` | **Entrada** de las Políticas de Peso — no una salida: peso, flete real, % kilo adicional, trayecto de tarifa resuelto + su genérico, tope de kilos |
| `DatosOriginalesParaProveedor` | Campos que el documento del proveedor dropshipper copia tal cual del documento de la tienda que vendió (réplica intencional del comportamiento legacy) |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-09-16 | Iker Acevedo | Documentación inicial de los modelos de dominio. |
| 2026-09-16 | Iker Acevedo | `Trayectos`/`IncrementosPeso` cambiados de `IReadOnlyList<T>` a `List<T>` — Newtonsoft no puede poblar una `ReadOnlyCollection` al deserializar directo desde Mongo (crash real visto en CloudWatch). Ver commit `892e372`. |
