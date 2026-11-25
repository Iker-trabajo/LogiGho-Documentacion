# Cobro de servicios

**Dominio:** cobro-servicios (spec 019 v2 y A23). **Lambda:** `ApiLambdaCobroServicios` (.NET 10, la misma atiende la API y la corrida). **Estado:** en PreProd probado por QA; **producción aún sin desplegar** (se hace con OK por paso, ver [Pasos para producción](#pasos-para-produccion)).

Cobra a las tiendas los servicios de proveedores externos y lo descuenta del saldo (liquidación) de la tienda. Hay dos servicios hoy:

| Servicio | Esquema | Cómo cobra | A quién |
|---|---|---|---|
| **Waichat** | `PaquetesPorRango` | Por guía según el rango (R1, R2, R3…) que alcanza el volumen del periodo; cargos únicos (API, Set up) al abrir; reclasificación al cerrar el mes | Contratos de **tienda** o de **ecosistema** (suma de las tiendas o por tienda) |
| **Pago a proveedores** (A23) | `PorcentajeSobreRecaudo` | Un porcentaje del `TotalRecaudo` de cada guía (0,04 % por defecto; 1 a 100.000 millonésimas). Sin rangos ni cargos únicos | Contrato de **Plataforma**: cubre todas las tiendas, activas o no, y las futuras |

## Cómo funciona

1. Una corrida (Step Function cada cierto tiempo, o a mano) lee los contratos `Activo` y registra en `CobrosServiciosDetalleGuias` las guías nuevas de `PedidosInter` con fecha de creación ≥ `FechaInicio` del contrato (**sin retroactivo**).
2. Cada guía pasa por estados de cobro: `Pendiente` → `Confirmada` (al entregarse, o al devolverse según `CobraEn`) / `Vencida` (45 días) / `PorValidar` / `Excluida`. Un débito al confirmarse; un abono si deja de corresponder.
3. Los movimientos van a `CobrosServiciosLogigho` (positivo = débito, negativo = abono) y la liquidación de la tienda los resta.
4. Al terminar el periodo (mes calendario o ciclo desde el inicio) se cierra: Waichat reclasifica al rango final y ajusta; Pago a proveedores solo cierra el mes.

Una guía se cobra en **un solo contrato** del mismo servicio (índice `ux_servicio_pedido`). Un contrato de tienda o de ecosistema gana sobre el de plataforma; lo que la plataforma ya registró no pasa al contrato de tienda creado después.

### Reglas de Pago a proveedores (A23)

- **Base:** `PedidosInter.TotalRecaudo` (texto en producción). Vacío, no numérico o 0 → guía `Excluida` con motivo `SinRecaudo`, sin movimiento.
- **Redondeo por guía** al peso, 0,5 hacia arriba, guardado en `PrecioCobrado`. Con 0,04 %: 96.800 → 39, 100.000 → 40, 1.250 → 1, 1.249 → 0. Con 0,4 %: 96.800 → 387, 100.000 → 400.
- **Sin contraentrega no se cobra** (`AplicaContraPago` distinto de `SI` → `Excluida / SinContraPago`). Es un parámetro del servicio (`CobrarSinContraPago`, apagado) que **solo CEO o Desarrollador** cambian; queda en el historial.
- `CobraEn`: Entrega (por defecto), Devolución o Ambos.
- Una guía cuyo `Idtienda` no existe en `Tienda` se omite con aviso.

## Colecciones (`LogighoDB`)

| Colección | Qué guarda |
|---|---|
| `ServiciosExternos` | Catálogo: `waichat` (con tarifarios Estándar y Ecosistemas) y `proveedores` |
| `ContratosServicioTienda` | Contratos (alcance Tienda, Ecosistema o Plataforma), periodos, cursores y progreso |
| `CobrosServiciosLogigho` | Movimientos de dinero |
| `CobrosServiciosDetalleGuias` | Una fila por guía con su `EstadoCobro` y `PrecioCobrado` |
| `CobrosServiciosCorridas` | Una fila por corrida: estado, contadores y bloqueos |
| `CobrosServiciosConfiguracion` | Candado de corrida y lista de estados que cobran |

Índices únicos obligatorios (si falta alguno la corrida no escribe): `ux_clave`, `ux_contrato_pedido`, `ux_servicio_pedido`, `ux_alcance_servicio_activo`.

## API (`/cobro-servicios/...`, REST de siempre)

PreProd `8u5isw5u30`, producción `7elgyten88`. Header `Token`. Respuesta `{ exito, datos, error }`; dinero en pesos COP enteros. Roles de administración: `Controller`, `CEO`, `Administrador`, `Desarrollador` (variable `ROLES_COBRO_SERVICIOS`).

| Ruta | Quién | Para qué |
|---|---|---|
| `GET/POST /servicios`, `PUT /servicios/{id}` | admin | Catálogo y tarifarios |
| `GET/POST /contratos`, `GET/PUT /contratos/{id}` | admin (el detalle también la tienda participante) | Contratos |
| `POST /contratos/{id}/estado` | admin | Activar, pausar, finalizar |
| `GET /contratos/conflictos` | admin | Detecta choques de alcance antes de crear |
| `GET /contratos/{id}/guias`, `POST /contratos/{id}/guias/finalizar` | admin (lista también la tienda) | Guías del contrato y finalizar a mano (A21) |
| `GET /guias-por-validar` | admin | Guías `PorValidar` y vencidas: Cobrar / No cobrar (A22) |
| `GET /movimientos`, `GET /movimientos/{id}/guias` | admin o tienda dueña | Movimientos de dinero |
| `POST /simular` | admin | Simulador de costos |
| `GET /ecosistemas` | admin | Ecosistemas disponibles |
| `GET/PUT /configuracion/estados` | admin / solo CEO y Desarrollador | Estados que cobran |
| `GET /mis-cobros` | cualquier usuario con sesión | Una fila por (contrato, tienda del usuario); la tienda solo ve lo suyo |
| `POST /corridas` | admin, **solo PreProd** | Corrida manual (403 `CORRIDA_MANUAL_DESHABILITADA` en producción) |

Errores frecuentes: 400 `VALIDACION` con códigos en `detalles` (`PORCENTAJE_INVALIDO`, `RANGO_SIN_CARGO_POR_GUIA`, `PLATAFORMA_REQUIERE_ESQUEMA_PORCENTAJE`…), 403 por rol, 409 `CONTRATO_ACTIVO_DUPLICADO` o contrato ya facturado.

## Frontend

El menú sale de la colección `Modulo`; las pantallas se abren por la ruta del módulo.

| Pantalla | Ruta | Módulo | Quién |
|---|---|---|---|
| **Mis servicios** | `/app/tienda/mis-servicios` | ID 172 (padre «Mi Tienda»); también el botón «Ver cobros de servicios» en Wallet | Tienda, Administrador, Controller, CEO, Desarrollador. La tienda ve solo sus contratos y cobros |
| **Cobro de servicios** (administración) | `/app/controller/cobro-servicios` | ID 173 (padre «Controller») | Controller, CEO, Administrador, Desarrollador (`cobroServiciosGuard`; el backend es quien autoriza de verdad) |

Mis servicios: contratos y cobros con filtros, paginación y totales del servidor, vista por tienda, guías del mes y resumen del porcentaje. Administración: servicios y tarifarios (incluye «Pago a proveedores» y el editor de rangos), contratos (tienda, ecosistema o toda la plataforma), detalle de contrato (periodos, movimientos, guías, reparto por tienda), simulador, estados de cobro, guías por validar y el interruptor de contraentrega (CEO y Desarrollador; Controller lo ve deshabilitado).

La ruta `cobro-servicios/` está en `RUTAS_REST` del `ConsumoGenericoService`; sin ella las llamadas irían al HTTP API y darían 404.

**Cambio vecino:** el retiro de las tiendas ya no resta comisión bancaria (`Monto a Consignar` = `Monto`).

## Cómo correr una corrida a mano

Evento normal `{"origen":"step"}`. La lambda toma un candado (`Omitida` si otra corrida lo tiene: no repetir el comando) y por cada contrato reconcilia, verifica el cuadre, registra guías, abre periodos, re-evalúa, mueve dinero, reparte y cierra. Con `fechaReferencia` (solo PreProd, `PERMITIR_FECHA_REFERENCIA=true`) se simula otra fecha: adelanta el «ahora» y cierra periodos antes de tiempo, úsala solo para pruebas. En producción esa variable debe estar en `false`. Detalle paso a paso en `specs/019_cobro_servicios/runbook-corrida-manual.md`.

Re-evaluación por lotes con punto de reanudación (M-342): la lambda no carga todos los detalles de los periodos abiertos en memoria; si una corrida no alcanza, continúa en la siguiente sin perder avance.

## Pasos para producción

Producción está limpia (sin lambda ni colecciones de cobro). Se hace con OK de Iker en cada paso, en este orden (runbook `specs/019_cobro_servicios/despliegue-prod.md`):

1. Infraestructura: `terraform plan` de `environments/prod/cobro-servicios` revisado (solo agrega: lambda, rol, log group, alarmas, rutas; 0 destrucciones) y apply.
2. Despliegue de la lambda desde `main` (CD de producción).
3. Módulos 172 y 173 en `Modulo` de producción (confirmar antes el ID máximo).
4. Servicio Waichat con sus tarifarios y contratos negociados, y servicio `proveedores` con el contrato de Plataforma.
5. Front en producción (`preprod` → `main`).

## Verificación

QA con datos reales copiados de producción a PreProd: veredicto **APTO** para Waichat y A23 (casos C03–C15 y V-A23; plan en `specs/019_cobro_servicios/plan-pruebas-e2e-v2.md`, §21 y §22). El valor de ese veredicto está atado a la versión de la lambda y del front con que se probó.
