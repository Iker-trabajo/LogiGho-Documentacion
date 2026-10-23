# Inventario — llamadores y estándar de llamadas

**Autor:** Iker Acevedo · **Fecha de creación:** 2026-10-21 · **Estado:** revisado en código el 2026-10-19; medido en PreProd (E2E 2026-10-19/20)

El día del go-live **las rutas no cambian de nombre ni de contrato**: solo cambia qué lambda las atiende. Ningún llamador necesita modificarse para seguir funcionando. Lo que sí cambia es el **comportamiento** (ver [abajo](#que-cambia-para-los-llamadores)).

Se encontraron **11 llamadores reales**: 4 de backend y 7 puntos del front. No hay más llamadores en el código del workspace.

## Llamadores de backend

| Llamador | Ruta | Cuándo llama | Qué manda | ¿Espera la respuesta? |
|---|---|---|---|---|
| **ApiLambdaCargarGuias** | `actualizaInventarioLotes` | Al terminar de crear las guías de una carga | `{ "Idcarga": "<id>" }` | No (fire-and-forget) |
| **ApiLambdaLiquidacionesLogighoAOT** | `actualizaInventarioLotes` | Cada 3 h tras liquidar (en producción se midió ~1 vez al día, ~12:00, con 1.700–1.900 filas) | `{ "Guias": [...], "IgnorarMarcaProcesado": true }` con `Token` | Sí; un fallo solo se registra en el log |
| **ApiLambdaDevolucionesMasivo** | `actualizaInventarioLotes` | Al terminar de marcar un lote de devoluciones escaneadas en bodega | `{ "Guias": [≤500] }` con cabecera `headersecurity`, **sin** `Token` | Sí; un fallo no tumba el job |
| **ApiLambdaConsumoSigueTuEnvio** | `actualizarInventario` | Cuando Inter indica devolución y el pedido ya quedó en «Devolución completada» | `{ "Guia": "<guía original>", "Operacion": 1 }` con `Token` | Sí; reintenta hasta 4 veces |

## Llamadores del front

| Pantalla (componente) | Ruta | Qué manda |
|---|---|---|
| Anular guías (Excel masivo) | `actualizaInventarioLotes` | `{ "Guias": [...], "Estado": "Anulada" }`; 3 intentos; avisa «inventario PENDIENTE» si fallan |
| Tabla de pedidos (anular una guía) | `actualizaInventarioLotes` | `{ "Guias": ["…"], "Estado": "Anulada" }`; si falla solo hay `console.warn` |
| Modal de creación de pedidos | `actualizaInventarioLotes` | `{ "Guias": ["<guía>"] }` tras generar la guía |
| Importación de inventarios (Excel de ingresos) | `actualizaInventarioLotes` | `{ "Ingresos": [{ "IdProducto", "Cantidad" }] }` en lotes de 50 |
| Creación masiva de productos | `actualizaInventarioLotes` | `{ "Ingresos": [ ... ] }` por producto, en serie |
| Creación de producto | `actualizaInventarioLotes` | `{ "Ingresos": [ ... ] }` por producto o variación |
| Inventario (modal de ajustes) | `actualizaInventarioLotes` | `{ "Ajustes": [{ "IdProducto", "Cantidad" }] }` (no manda `Fecha`; el servidor pone la hora actual) |

**No son llamadores** (para no confundirse): las funciones `ActualizaInventario(...)` de los procesos automáticos de Inter, Envía, Servientrega y XCargo no llaman a ninguna ruta —restan directo de `Productos.cantidad`; solo existen las definiciones, es código del stock legado—. Tampoco llaman al inventario `gestion-guias`, `ApiLambdaAnularGuia`, `ApiLambdaActualizacionConciliaciones` ni `ProcesoGuiasArchivadas`; esa es la causa de buena parte de la suciedad histórica, y el [barrido](../../guias/inventario-operacion.md#barrido-de-cierre) cubre ese hueco.

El front `ingreso-devoluciones` llama a `devolucionesMasivo/iniciar`, que dispara el llamador de DevolucionesMasivo de forma indirecta.

## Estándar de llamadas

Basado en el E2E de PreProd (2026-10-19/20). Aplica a Liquidaciones, DevolucionesMasivo, CargarGuias, SigueTuEnvio, anular-guias, importación de ingresos y **cualquier proceso futuro**.

1. **Lotes de 1400 guías o menos por llamada.** Con ≈5 ms por movimiento, 1400 guías entregadas (2 movimientos cada una) tardan ≈13 s. El API Gateway corta a ~30 s (503): en la prueba, 500 guías → 3,9 s; 2000 → 19,1 s; **3500 → 503 a los 30,6 s** (la lambda terminó a los 33,6 s y dejó todo bien).
2. **En serie, o como máximo 2 a la vez.** Tres lotes paralelos de 600 tardaron 14–18 s cada uno: **sin ganancia** frente a hacerlos en serie (el cuello es la base de datos). El límite de lambdas concurrentes de la cuenta en PreProd es 10.
3. **Un `RecalcularResumen` al final** de una tanda con varios lotes: dos lotes simultáneos del mismo producto recalculan su resumen a la vez y un recálculo viejo podría pisar a uno nuevo (no se vio en la prueba, pero es posible).
4. **Reintentar un 503/504 es seguro para pedidos** (idempotente; 0 duplicados medido), pero la lambda sigue trabajando: **esperar unos segundos** antes de reintentar y no lanzar reintentos en paralelo. Cada reintento con 3500 guías costó ≈6 s de lectura.
5. **Ingresos y ajustes NO son idempotentes.** Nunca reintentar a ciegas tras un 503/504, nunca en paralelo sin control; deshabilitar el botón mientras se envía y verificar en la trazabilidad antes de repetir.
6. **No depender del texto de la respuesta.** Solo importa el código HTTP.
7. **Un cambio de estado posterior a un cierre no se ve** desde Liquidaciones, DevolucionesMasivo ni SigueTuEnvio: con `EntregasInventarioProcesado=SI` ya no releen la guía. Un proceso que cambie el estado de una guía ya cerrada debe avisar con **ambas** banderas ignoradas (`IgnorarMarcaProcesado` e `IgnorarMarcaEntregasProcesado` en `true`) o dejar que lo corrija el **barrido**, que debe estar activo.
8. **No llamar al inventario si la guía no se creó.** El modal de creación lo hace aunque la transportadora falle, y manda el texto del error como número de guía.

### Resumen por llamador

| Llamador | Lote máximo | Notas |
|---|---|---|
| Liquidaciones | 1400 en serie | **Hoy sin tope: limitar** (pendiente, ver [Tareas futuras](tareas-futuras.md#partir-lotes-en-liquidaciones)) |
| DevolucionesMasivo | 500 (ya) | Sin `Token`, con `headersecurity` |
| CargarGuias | `Idcarga` | Lee por dentro de 500 en 500 |
| SigueTuEnvio | 1 guía | Con `Token` |
| anular-guias | 1400 en serie | Hoy un solo POST con todo el Excel |
| importacion-inventarios | 50 por lote, **no** paralelo | Ingresos no idempotentes |

### Quién es el dueño del inventario

La activación vive en el **proveedor**: `Tienda.ServicioInventario=true` en la tienda dueña (`Productos.IdTienda`) y `Productos.ManejaInventario=true` en cada producto. La tienda **vendedora** (dropshipper) no necesita nada.

## Qué cambia para los llamadores

| Tema | Lambdas viejas | Sistema nuevo | Quién lo nota |
|---|---|---|---|
| **Filtro de activación** | Cualquier producto que existiera en `Productos`, incluso inactivos | Solo productos activados de tiendas con servicio, desde su primer ingreso. Ingresos y ajustes se escriben siempre | Todos los de pedidos |
| **Auto-ingreso** | Creaba un ingreso desde `Productos.cantidad` | Eliminado: producto sin ingresos = 0 | CargarGuias, Liquidaciones, DevolucionesMasivo, SigueTuEnvio, anulaciones |
| **Fórmula** | `Consumo = Ingresos − Actual` | v2: el ajuste solo mueve `Actual` | Pantalla de ajustes y las que leen el resumen |
| **Un cierre por guía** | Podía haber varios | Uno por guía y producto; la evidencia física gana | Liquidaciones, DevolucionesMasivo, SigueTuEnvio, anulaciones |
| **«Devolución completada»** | Solo el texto exacto sin tilde (766 guías BTK con tilde se perdían) | Se normaliza tilde, mayúsculas y espacios | DevolucionesMasivo, SigueTuEnvio |
| **Retorno con cierre ya registrado** | Se borraba o dejaba sin criterio | Se conserva y se marca | Liquidaciones y barrido |
| **Pérdidas** | No existían | `PerdidasInventario` | Liquidaciones y barrido |
| **Estado desconocido** | Se ignoraba | `Revisar`: no se toca y se cuenta | Todos |
| **Idempotencia** | «Consulta y luego inserta» | Índice único; `E11000` = ya existía | Reintentos de SigueTuEnvio y anular-guias |
| **Lectura de pedidos** | Todo de una vez | De 500 en 500; sin `Idcarga` ni guías no lee nada | CargarGuias, Liquidaciones |
| **Banderas** | Se marcaban a mitad de camino | Se marcan al final | Todos |
| **Respuesta** | `Operación exitosa.` / `Operacion Exitosa` | `Operación exitosa.` o `Sin cambios: …` | Nadie la lee |
| **Cuerpo sin guía, carga ni ingresos/ajustes** | `PorGuia` 200 con mensaje; `PorCarga` 400 | 400 en ambas | Ningún llamador lo manda |
| **Errores 500** | Con `e.Message` | Genérico (detalle en el log con `correlationId`) | — |
| **CORS** | `*` con credenciales | `*` sin credenciales (decisión de Iker) | Las 7 pantallas del front |
| **Historial** | No había | Todo borrado o corregido deja el original | Soporte |

## Riesgos conocidos por llamador

!!! warning "Pendiente: nada de esto se midió contra producción"
    Las cifras de tiempo vienen de PreProd (Atlas). El ensayo con DocumentDB de producción es parte de la ventana.

- **Lotes grandes (Liquidaciones, DevolucionesMasivo, anular-guias, importación).** Si pasan de ~30 s el llamador recibe 503/504 aunque los datos queden bien; Liquidaciones y SigueTuEnvio reintentan hasta 4 veces y cada reintento lanza otra ejecución sobre las mismas guías. El índice único evita los duplicados; el costo es carga extra. Decisión tomada: **aceptar en el go-live** (el volumen medido de Liquidaciones cabe en 30 s) y partir en lotes de ≤1400 después ([Tareas futuras](tareas-futuras.md#partir-lotes-en-liquidaciones)).
- **Importación de ingresos con 504.** Si el usuario vuelve a importar, los ingresos se duplican (no son idempotentes). Hay que medir el recálculo con 50 productos distintos por lote.
- **Authorizer.** `/actualizarInventario` lleva Cognito en producción: un token vencido en SigueTuEnvio da 401 y el movimiento se registra más tarde (liquidación o barrido). `/actualizaInventarioLotes` **debe seguir sin authorizer** o DevolucionesMasivo (que no manda `Token`) se rompe.
- **Anular desde la tabla.** Si falla, nadie se entera y la guía queda sin anulación hasta que el barrido la corrija. La liquidación excluye las anuladas, así que solo el barrido cubre ese hueco.
- **Productos nuevos.** No están activados, así que sus pedidos no descuentan inventario hasta activarse con *Administrar inventario* o script.
- **Cargas de guías en PreProd.** `ApiLambdaCargarGuias` de PreProd **no está aislada de Interrapidísimo**: llamó al servicio real y creó 14 preenvíos reales; por eso el camino CargarGuias → inventario se probó por el puente y no de punta a punta. En la ventana de producción se hace una prueba de humo con una guía real ([Tareas futuras](tareas-futuras.md#aislar-preprod-de-inter)).
