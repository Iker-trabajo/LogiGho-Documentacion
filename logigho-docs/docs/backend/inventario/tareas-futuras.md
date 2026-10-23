# Inventario — tareas futuras

**Autor:** Iker Acevedo · **Fecha de creación:** 2026-10-21 · **Estado:** lista viva; nada de esto bloquea el go-live

Lo que quedó **fuera** de la liberación, con su motivo. Cada punto dice qué está confirmado y qué falta.

## Partir lotes en Liquidaciones

- **Qué:** `ApiLambdaLiquidacionesLogighoAOT` manda todas las guías liquidadas en una sola llamada a `actualizaInventarioLotes`, sin tope.
- **Por qué:** el API Gateway corta a ~30 s; 3500 guías ya rebasan (medido en PreProd). El estándar es ≤1400 en serie ([estándar](llamadores.md#estandar-de-llamadas)).
- **Estado:** el volumen medido hoy en producción (1 vez al día, ~1.700–1.900 filas, ~15–20 s) cabe. Un atraso de varios días no cabría. **Es código financiero: no se toca ahora** (decisión D2); se parte después del go-live.

## Validación de stock al crear guías

- **Qué:** el front valida contra `Productos.cantidad`; el inventario nuevo no lo toca (INV-030). En el E2E el front permitió pedir 610 u con `Actual` 496.
- **Propuesta:** `GET /inventario/disponible` + validación en servidor, bandera por tienda, empezando en «solo avisar».
- **Antes:** medir `Actual` tras la limpieza (el producto 10402 en producción está en −263 antes de limpiar).

## Ingresos y ajustes idempotentes

- **Qué:** son acciones manuales sin guía, no tienen clave natural: un doble clic o un reintento tras un 504 duplica (B-3).
- **Propuesta:** `IdSolicitud` opcional enviado por el cliente → `clave = ING|IdSolicitud|n` con el mismo índice único (front y back).
- **Mientras tanto:** deshabilitar el botón al enviar, no reintentar a ciegas, verificar en la trazabilidad.

## «Despachado» real

- Hoy «despachado» = salida registrada en el libro (día local), no el despacho físico. Propuesta: campo **`FechaDespachoInventario`** en el pedido, definido con el Jefe de Logística. Es un cambio localizado en `AgregadorDespachos` y su fuente ([Alertas](alertas-slack.md#que-es-despachado)). También habilita `TRANSITO_VENCIDO` sin fecha de despacho.

## Causas raíz de las anomalías

Origen de la suciedad histórica (hipótesis marcadas; análisis en el informe de anomalías de producción). El sistema nuevo la **detecta y corrige**, pero seguirá apareciendo mientras estos procesos no cambien:

| Causa raíz | Efecto | Estado |
|---|---|---|
| **`ProcesoGuiasArchivadas`** (y `ProcesoAutomaticoInter`) voltean `Archivada → Devolución ratificada` y vacían `Fecha Entrega` según el **orden** de los eventos del historial de Inter | Entregas reales quedan en un estado de retorno (25 ENT y 30 DEV en BTK) | Confirmado en código; falta decidir por **fecha más reciente** |
| **`gestion-guias`**: `toCanonicalEstado` mapea «devolución completada» a `Devolucion` (degrada el estado) y devuelve a «Creada» sin reversar la anulación | 284 guías `Devolucion` con DEV en BTK | **Hipótesis por confirmar** con datos reales |
| **Anular y conciliar no avisan al inventario** (`ApiLambdaAnularGuia`, `ApiLambdaActualizacionConciliaciones`); la liquidación excluye las anuladas | Anulaciones y pagos sin cierre registrado | Confirmado; lo cubre el barrido |
| **Guías `800000000xxx` repetidas** (hasta 13 repeticiones, 120 guías) | Pedidos repetidos que el sistema omite | Origen **sin encontrar** en el repo (¿carga masiva o creación manual?). Falta unicidad de `Numeropreenvio` y limpiar `PedidosInter` |
| Comparación exacta de «Devolucion Completada» en las lambdas viejas | 766 guías BTK con tilde nunca se registraban | Resuelto por el sistema nuevo |

Mientras `gestion-guias` y los procesos de archivadas no se corrijan, el **barrido debe quedar en simulación** (riesgos R-1…R-4 del informe).

## Aislar PreProd de Inter

- **Qué:** `ApiLambdaCargarGuias` en PreProd llamó al servicio real de Interrapidísimo (**14 preenvíos reales**, vencen 2027-01-19; hay que anularlos con Inter) y falló al escribir en el bucket de guías de **producción** (`Access Denied`).
- **Consecuencia:** CargarGuias → inventario no se probó de punta a punta; en la ventana se hace humo con una guía real.
- **Tarea de plataforma:** que PreProd use sandbox de la transportadora y su propio bucket.

## Access logs del API

- Sin access logs en el HTTP API puente, no queda el código HTTP de cada llamada. **Activarlos** también en las rutas de inventario del REST de producción (checklist previo a la ventana).

## Notificador en producción

- El módulo ya soporta `notificador_slack_arn`; en producción está vacío porque no se verificó que `ApiLambdaNotificadorEventos` exista allí. Hoy las 22 alarmas llegan por correo. Verificar el notificador y conectarlo.

## Retirar el puente `g3iz6qk3f0`

Cuando la rama de infraestructura incorpore `main` (con novedades): 1) `apply` de `environments/preprod/datos` (ya trae trazabilidad, escritura y alertas en el REST `8u5isw5u30`) con la cadena real; 2) apuntar los llamadores y el front de PreProd al REST; 3) `puente_g3iz6qk3f0.ps1 -Quitar`; 4) devolver la URL de humo de `cd-preprod.yml` a `https://8u5isw5u30.execute-api.us-east-1.amazonaws.com/PreProd`. **No aplicar `datos` desde la rama actual:** destruiría el módulo `novedades_webhook`.

## Otras mejoras abiertas

| Tema | Detalle |
|---|---|
| Barrido en escritura | Tras una semana de simulación limpia y decidida la regla de retorno (D1) |
| Historial | ~622 mil entradas `SELLO` (~317 MB) en el go-live. Recomendación D3: **no guardarlas** (el respaldo JSON ya tiene los originales); si se guardan, índice TTL de 365 días |
| Admisión de devoluciones | `PendienteAdmision` es un tope superior: no hay forma de marcar una devolución como admitida |
| Activación sin pantalla completa | Productos nuevos solo se activan con *Administrar inventario* o script |
| `Tienda.ZonaHoraria` | Hoy una sola zona (`America/Bogota`) para leer `PedidosInter` (INV-026) |
| Catálogo único de estados | Consolidar las tres colecciones de homologación |
| Parámetros | `DiasAlertaRetornoVencido`, `DiasAlertaTransitoVencido`, `UmbralBajoStock` en `ParametrosGenerales` |
| Roles | Centralizar constantes de rol y retirar `'Lider de Logistica'` (INV-005) |
| Cifrado del webhook | Pasar de AES-ECB a GCM o KMS |
| Código legado | Retirar las funciones `ActualizaInventario` de los procesos automáticos (falsas pistas) |
| Textos | La trazabilidad dice «docs/u»: cambiar a «registros/unidades» |
| Gemelas | Apagar y borrar `ApiLambdaInventarioCarga` y `ApiLambdaActualizacionInventario` tras la semana de observación |
| Pruebas del front | Línea base de ~90 fallos «should create» ajenos al inventario |
