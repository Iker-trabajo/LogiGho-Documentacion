# Inventario — visión del dominio

**Autor:** Iker Acevedo · **Fecha de creación:** 2026-10-21 · **Estado:** liberación a producción planeada; validado en PreProd

El inventario de LogiGho se reemplazó por un sistema nuevo, **`ApiLambdaInventario`** (.NET 10, una base de código, 8 funciones Lambda). Esta sección documenta cómo funciona, cómo se llama, cómo se opera y qué queda pendiente.

!!! info "Cómo leer el estado de cada afirmación"
    - **Confirmado** = está en código, en Terraform o en un reporte de validación (VAL-1/2/3, ensayo de go-live, E2E en PreProd).
    - **Pendiente** = decidido o propuesto, pero no hecho, no medido o no verificado. Siempre se dice como tal.
    - A la fecha de esta página **nada del sistema nuevo está desplegado en producción** (solo existen allí los índices `ux_clave` y los de trazabilidad). La salida a producción sigue el [runbook de operación](../../guias/inventario-operacion.md).

## Mapa de esta sección

| Página | Qué responde |
|---|---|
| **Esta página** | Qué problema había, qué se construyó, glosario |
| [Arquitectura](arquitectura.md) | Funciones, flujo por guía, diagrama, Strangler Fig, filtro de activación |
| [Estados, cierres y fórmula](estados-y-formula.md) | Homologación de estados, un cierre por guía, fórmula v2 con ejemplo |
| [API de inventario](../../api/inventario.md) | Contratos de las rutas, errores, seguridad, CORS |
| [Llamadores y estándar de llamadas](llamadores.md) | Quién llama a las rutas y cómo debe hacerlo |
| [Alertas diarias por Slack](alertas-slack.md) | Producto nuevo de suscripciones |
| [Operación (runbook)](../../guias/inventario-operacion.md) | Go-live, rollback, observabilidad, barrido, invariantes |
| [Decisiones (ADR)](../../arquitectura/adr/inventario/index.md) | Por qué se eligió cada diseño |
| [Tareas futuras](tareas-futuras.md) | Lo que quedó fuera de la liberación |
| [Pantallas del front](../../frontend/views/logistica/inventario/inventario-front.md) | Angular que consume el sistema |
| [Changelog de la liberación](../../changelog/inventario-2026-10.md) | Qué cambia el día del go-live |

## El problema

El inventario anterior eran dos Lambdas **gemelas** con la misma lógica copiada: `ApiLambdaInventarioCarga` (ruta `actualizaInventarioLotes`) y `ApiLambdaActualizacionInventario` (ruta `actualizarInventario`). Los defectos medidos en la auditoría del ecosistema BTK:

| Síntoma | Causa |
|---|---|
| Salidas y devoluciones **duplicadas** | La lambda preguntaba «¿ya existe?» y luego insertaba, en dos pasos separados. Dos llamadas casi simultáneas (reintentos tras un 504, la liquidación y una carga a la vez) veían «no existe» y ambas insertaban |
| **Tránsito negativo** | `Consumo = Ingresos − Actual`: los ajustes de bodega se mezclaban con el consumo de pedidos |
| Guías eternamente «en tránsito» | Solo la liquidación (cada 3 h) registraba cierres; los procesos que cambian `PedidosInter.Estado` (transportadoras, archivadas, gestión de guías, conciliaciones, anulaciones) no avisaban al inventario |
| Devoluciones que nunca se registraban | Solo se reconocía el texto exacto `Devolucion Completada` (sin tilde); había 277 variantes de estado |
| Anulada **y** entregada a la vez | Nadie decidía cuál era el cierre verdadero; se sumaban los dos |
| Stock creado de la nada | Auto-ingreso desde `Productos.cantidad` |
| Cada arreglo, dos veces | Lógica duplicada en las gemelas, que ya habían divergido |

## La solución

1. **Una sola base de código** con varios handlers; cada handler es su propia función Lambda ([Arquitectura](arquitectura.md)).
2. **`PedidosInter` es la fuente de verdad.** Para cada guía el sistema calcula qué movimientos *deberían* existir y compara con los que *existen*; escribe solo la diferencia ([flujo por guía](arquitectura.md#flujo-por-guia)).
3. **Idempotencia real:** cada movimiento lleva una `clave` única (`guía|producto|casilla|TIPO`) con índice único + sparse. Repetir o simultanear una escritura no puede duplicar ([ADR-INV-001](../../arquitectura/adr/inventario/ADR-INV-001-clave-unica-e-indice-sparse.md)).
4. **Un solo cierre por guía y producto**, decidido por el estado homologado del pedido ([ADR-INV-002](../../arquitectura/adr/inventario/ADR-INV-002-un-cierre-por-guia.md)).
5. **Fórmula v2:** los ajustes solo mueven lo que hay en bodega, no el tránsito ([ADR-INV-003](../../arquitectura/adr/inventario/ADR-INV-003-formula-v2.md)).
6. **Solo se gestiona lo activado:** filtro de activación por proveedor ([ADR-INV-004](../../arquitectura/adr/inventario/ADR-INV-004-filtro-de-activacion.md)).
7. **Migración gradual (Strangler Fig):** las rutas se repuntan a las funciones nuevas sin que los llamadores cambien nada, y el rollback es repuntar de vuelta.
8. **Trazabilidad, historial y barrido** para ver y corregir lo que los procesos externos dejan descuadrado.
9. **Producto nuevo:** [alertas diarias por Slack](alertas-slack.md).

## Glosario

| Término | Significado |
|---|---|
| **Guía** | Número de envío (`Numeropreenvio` en `PedidosInter`). Un pedido = una guía, con hasta 30 casillas de producto |
| **Casilla / slot** | Par `IdStockN` / `CantidadStockN` del pedido. Un mismo producto en dos casillas (por ejemplo, dos tallas) son dos movimientos legítimos |
| **Salida** | Registro (`SalidaInventario`) de que unidades de un producto salieron de bodega por un pedido. **Incluye las guías anuladas**: la anulación las devuelve después |
| **Anulación** | Cierre (`AnulacionesInventario`): la guía nunca salió físicamente; las unidades **vuelven** a `Actual` |
| **Entrega** | Cierre (`EntregasInventario`): el destinatario recibió. Incluye las guías **Pagada** aunque no tengan `Fecha Entrega` |
| **Devolución** | Cierre (`DevolucionesInventario`): la guía llegó de vuelta («Devolución completada»). Las unidades **no** suman a `Actual` solas: el ingreso a bodega lo hace una persona tras validar la mercancía |
| **Pérdida** | Cierre nuevo (`PerdidasInventario`): indemnizada, siniestrada, facturada o con error. Cierra el tránsito pero **no** devuelve las unidades a bodega |
| **Cierre** | Cualquiera de los cuatro movimientos finales de una guía: entrega, devolución, anulación o pérdida. Solo puede existir **uno** por guía y producto |
| **Tránsito** | Unidades que salieron y aún no tienen cierre. Se desglosa en: por despachar, admitida, tránsito de ida, tránsito de retorno y sin clasificar |
| **Retorno** | La guía vuelve a bodega (estado «Devolución», «En devolución», «Devolución ratificada»…) pero aún no es «Devolución completada» |
| **Pendiente de admisión** | Devoluciones registradas que aún esperan el ingreso manual a bodega. Queda fuera del tránsito |
| **Sello** | Operación de la limpieza: agregar `slot` y `clave` a un documento viejo sin cambiar su significado |
| **Clave** | `guía|producto|casilla|TIPO` (tipos `SAL`, `ENT`, `DEV`, `ANU`, `PER`). Base de la idempotencia |
| **Guías generadas** | Nombre visible en Slack del mensaje que antes se llamaba «despachos del día»: guías con salida registrada ese día local menos anulaciones del día |
| **Barrido** | Función que recorre las guías con movimientos y reconcilia las que quedaron descuadradas por procesos que no avisan al inventario |
| **Alcance BTK** | Primer alcance del sistema: ecosistema BTK y tiendas propias, 65 productos activos (INV-001, INV-028) |

## Alcance y límites

!!! success "Confirmado"
    - Alcance inicial: productos **activos** del ecosistema BTK (65 productos). Los inactivos no se limpian ni se sincronizan.
    - Validado en PreProd sin fallos del inventario: VAL-1, VAL-2, VAL-3, ensayo completo del go-live (37 pasos, rollback incluido) y E2E (21 casos).

!!! warning "Pendiente o fuera de alcance"
    - Producción: nada desplegado. La ventana se ejecuta con el [runbook](../../guias/inventario-operacion.md).
    - Ingresos y ajustes manuales **no son idempotentes** (igual que antes).
    - El barrido arranca en **simulación**: los cierres de guías en retorno no se tocan hasta decidir la regla de escritura.
    - Lista completa en [Tareas futuras](tareas-futuras.md).
