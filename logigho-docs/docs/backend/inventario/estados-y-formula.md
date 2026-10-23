# Inventario — estados, cierres y fórmula

**Autor:** Iker Acevedo · **Fecha de creación:** 2026-10-21 · **Estado:** reglas aprobadas por Iker; validadas en PreProd

## Homologación de estados

`PedidosInter.Estado` es texto crudo de cada transportadora (277 variantes distintas en producción, con ciudad incluida: «En novedad por Cali»). `ClasificadorEstado` lo lleva a **una categoría de inventario**, normalizando tildes, mayúsculas y espacios. Un estado desconocido **nunca se adivina**: pasa a `REVISAR`.

| Categoría | Quién tiene el paquete | Estados (patrón) | Efecto en inventario |
|---|---|---|---|
| **POR_DESPACHAR** | Remitente | Creado/Creada, Cargado, «Generada en X», En alistamiento | Tránsito: por despachar |
| **ADMITIDA** | Transportadora, aún sin rodar | Admitida/Recogida, En procesamiento, En bodega origen | Tránsito: admitida |
| **TRANSITO_IDA** | Transportadora, en movimiento | Tránsito, Centro de acopio, Reparto, Novedad, Reenvío, Reexpedición, Intento de entrega… | Tránsito: ida |
| **ENTREGADA** | Destinatario | Entregada, **Pagada** (aunque no tenga `Fecha Entrega`) | Cierre **Entrega** |
| **RETORNO** | Transportadora, de vuelta | Devolución, Devolución ratificada, En devolución, Entregado a remitente | Tránsito: retorno (alerta si vence) |
| **DEVOLUCION_COMPLETADA** | Bodega o remitente | Devolución completada (con o sin tilde, con espacios) | Cierre **Devolución** → pendiente de admisión |
| **ANULADA** | Remitente (nunca salió) | Anulada | Cierre **Anulación** (la unidad vuelve a `Actual`) |
| **PERDIDA** | Nadie (reclamo a la transportadora) | Indemnizada, En proceso de indemnización, Siniestrada, Destrucción/salvamento, **Facturado**, **Error** | Cierre **Pérdida** (no vuelve a `Actual`) |
| **REVISAR** | — | Estado desconocido o ambiguo | No se toca nada; se cuenta |

Estados que se deciden **por fechas**:

| Estado | `Fecha Entrega` | `Fecha Devolucion` | Resultado |
|---|---|---|---|
| **Archivada / Digitalizada** (Inter) | sí | — | Entregada (9.856 pedidos) |
| | no | sí | Retorno |
| | ambas / ninguna | | **Revisar** (60 pedidos sin fechas) |
| **Devolución Regional** (Inter, [INV-039](../../arquitectura/adr/inventario/ADR-INV-005-pedidosinter-fuente-de-verdad-y-retorno.md)) | sí (con o sin la otra) | | Entregada |
| | no | sí | Retorno |
| | no | no | Tránsito de ida (y si ya tiene cierre, se conserva como retorno) |
| **Anulada con `Fecha Entrega`** | sí | | **Entrega**, no Anulación (evidencia física) |

!!! note "Catálogo único de estados"
    El diseño de fondo es que `CatalogoEstadosTransportadora` (iniciado para TCC) sea el catálogo de toda la plataforma. **Pendiente:** hoy conviven tres colecciones que se solapan (`CatalogoEstadosTransportadora`, `EstadoGuiasHomologacion`, `EstadoGuiaLogighoNew`) y el inventario clasifica **en código** con pruebas que cubren las 277 variantes reales. Cuando exista el catálogo, el inventario lo leerá sin cambiar su contrato. Los parámetros de umbral (`DiasAlertaRetornoVencido` 45, `DiasAlertaTransitoVencido` 30, `UmbralBajoStock` 1000) irán en `ParametrosGenerales` (`IdTienda=0` es el valor por defecto; una fila por tienda lo sobrescribe).

## Un cierre por guía

`ReglaCierreUnico` decide qué cierre debe existir para cada guía × producto ([ADR-INV-002](../../arquitectura/adr/inventario/ADR-INV-002-un-cierre-por-guia.md)):

| Estado del pedido | Salida | Cierre que debe existir |
|---|---|---|
| Por despachar, admitida, tránsito de ida | sí | ninguno |
| Entregada o **Pagada** | sí | Entrega |
| **Devolución completada** | sí | Devolución (siempre; si había Entrega sobrante, se **borra** con historial; en una Pagada con devolución gana la entrega, INV-035) |
| Anulada | sí | Anulación, **salvo** que traiga `Fecha Entrega`: entonces Entrega |
| Indemnizada, siniestrada, facturada, error | sí | Pérdida |
| **Retorno** | sí | ninguno nuevo; el cierre que ya exista **se conserva** |
| **Revisar** | no se toca | no se toca |
| Vuelve a «Creada» (reversión) | sí | se borra el cierre existente |

La **salida se conserva siempre** (modelo A: incluye las anuladas; la anulación las devuelve).

### Guías en retorno con cierre ya registrado (INV-038)

Si el pedido está en retorno (no «Devolución completada») y ya tiene un cierre, **manda `PedidosInter`**: la guía sigue en camino de vuelta. El reconciliador **no borra, no corrige ni inserta** ninguno de sus cierres (la salida sí se reconcilia). La trazabilidad lo marca:

- `RETORNO_PENDIENTE_BODEGA` si el cierre es una Devolución (espera «Devolución completada»).
- `ESTADO_CONTRADICTORIO` si es Entrega, Anulación o Pérdida (o una Devolución con otro cierre): **desarrollo revisa el estado del pedido**.

En el resumen, esas unidades cuentan en `TransitoRetorno` y `RetornoConCierre` dice cuántas son de este tipo. El criterio general: **con estado ambiguo nunca se destruye evidencia**. En el ensayo, de las 346 guías BTK en esta situación, 314 quedan como `RETORNO_PENDIENTE_BODEGA` (aceptadas) y 30 como `ESTADO_CONTRADICTORIO` (a desarrollo) ([ADR-INV-005](../../arquitectura/adr/inventario/ADR-INV-005-pedidosinter-fuente-de-verdad-y-retorno.md)).

### Otras reglas de reconciliación

- **Cantidades:** si el pedido se editó, lo guardado se corrige al valor del pedido (motivo `CANTIDAD_PEDIDO`). Casillas con cantidad ≤ 0 o ilegible se saltan.
- **Duplicados viejos** (sin `slot`/`clave`): se emparejan con las casillas; el más antiguo se conserva y se sella, el resto se borra (`DUPLICADO`).
- **Producto quitado de un pedido editado:** sus movimientos se borran (`PRODUCTO_FUERA_DEL_PEDIDO`) salvo que la casilla exista pero sea ilegible.
- **Guía repetida en `PedidosInter`** (mismo `Numeropreenvio` en varios pedidos): se omite y se cuenta (`PedidosRepetidos`); no se sabe cuál es la verdadera. Son las guías de la serie `800000000xxx` (ver [Tareas futuras](tareas-futuras.md#causas-raiz-de-las-anomalias)).
- **Motivos de historial:** `DUPLICADO`, `REVERSION`, `EVIDENCIA_FISICA`, `CIERRE_DISTINTO`, `PRODUCTO_FUERA_DEL_PEDIDO`, `CANTIDAD_PEDIDO`, `SELLO`.

## Fórmula v2

Aprobada por el CEO y el Jefe de Logística ([INV-036](../../arquitectura/adr/inventario/ADR-INV-003-formula-v2.md)); se sale a producción con `FORMULA_CONSUMO=v2`.

```
Actual     = Ingresos − Salidas + Anulaciones + Ajustes     ← lo que hay en la estantería
Consumo    = Salidas − Anulaciones                          ← lo que salió por pedidos
En tránsito = Consumo − Entregas − Devoluciones − Pérdidas  ← lo que está en la calle
```

Los **ajustes solo mueven `Actual`**. Las devoluciones **no** suman a `Actual` solas (ingreso manual tras validar la mercancía, INV-004). Se eliminó el auto-ingreso desde `Productos.cantidad` (INV-030): un producto sin ingresos muestra 0.

### Ejemplo numérico (producto 10402 del E2E en PreProd)

| Movimiento | Unidades |
|---|---|
| Ingresos | 1000 |
| Ajustes (conteo de bodega) | −3 |
| Salidas | 18 |
| Anulaciones | 1 |
| Entregas | 7 |
| Devoluciones | 3 |
| Pérdidas | 1 |

| Cálculo | Con v2 | Con v1 (fórmula anterior) |
|---|---|---|
| `Actual` = 1000 − 18 + 1 − 3 | **980** | 980 |
| `Consumo` | **17** = 18 − 1 | 20 = 1000 − 980 *(contaminado por el ajuste)* |
| `En tránsito` | **6** = 17 − 7 − 3 − 1 | 9 = 20 − 7 − 3 − 1 |

El ajuste de −3 baja `Actual` (983 → 980) pero **no mueve el tránsito**. VAL-3 lo probó: un ajuste de −3 en un producto de prueba bajó `Actual` de 78 a 75 y dejó `Consumo` y `En tránsito` igual.

### Desglose del tránsito

`PorDespachar`, `Admitida`, `TransitoIda`, `TransitoRetorno` y `SinClasificar` se calculan **guía por guía** sobre las guías con salida y sin cierre. `SinClasificar` reúne las unidades abiertas que ninguna cubeta explica (guía sin pedido, estado `Revisar`, o un estado que dice «cerró» sin su cierre registrado todavía). Como en retorno con cierre los cierres siguen restando, el desglose cuadra así:

```
EnTransito = PorDespachar + Admitida + TransitoIda + (TransitoRetorno − RetornoConCierre) + SinClasificar
```

`PendienteAdmision` (devoluciones registradas) queda **fuera** del tránsito. Hoy es un tope superior: no hay forma de marcar una devolución como admitida (pendiente, ver [Tareas futuras](tareas-futuras.md)).

!!! warning "Desfase conocido del desglose"
    El desglose del resumen (`TransitoRetorno`…) queda desactualizado tras un cambio de estado que no genera movimientos nuevos, hasta el siguiente recálculo (hallazgo H9 del E2E). El recálculo corre con el barrido y puede invocarse a mano.
