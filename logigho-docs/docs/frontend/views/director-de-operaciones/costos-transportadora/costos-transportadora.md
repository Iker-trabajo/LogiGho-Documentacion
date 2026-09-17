## Autor: Iker Acevedo
Fecha creación: 2026-09-16
Estado: preprod
Tipo: vista

# Costos Transportadora

**Selector:** `app-costos-transportadora`
**Ubicación:** `src/app/views/director-de-operaciones/costos-transportadora`
**Acceso:** Autenticado — roles `Director de Operaciones`, `CEO`, `Administrador`, `COO`, `Desarrollador`

---

## ¿Qué hace?

Es la pantalla donde se configura el [motor Dinámico](../../../../backend/lambdas-dotnet/lambdas/ApiLambdaLiquidacionesLogighoAOT/motor-dinamico/overview.md) de liquidaciones: qué transportadoras liquidan con el motor nuevo, qué política de peso usan, qué trayectos aceptan y cuánto cobra cada tienda por cada uno. Antes de este módulo, migrar una transportadora al motor Dinámico significaba editar documentos de Mongo a mano; ahora es un formulario con validación, igual que cualquier otra pantalla de la plataforma.

No reemplaza nada del sistema legacy de tarifas (**Costos Transporte Tienda**) — las transportadoras que siguen en el motor Legacy (Interrapidísimo, Envía, Servientrega) siguen configurándose ahí. Este módulo es exclusivamente para transportadoras migradas o en proceso de migrar al motor Dinámico.

---

## Dos vistas, sin cambio de URL

El componente maneja internamente dos pantallas usando `history.pushState` (para que el botón atrás del navegador funcione sin salir del componente — mismo patrón que `GestionDevolucionesComponent`):

1. **Hub** — selector de transportadoras con cards. Muestra las 4 transportadoras conocidas (TCC, Interrapidísimo, Servientrega, Envía); solo TCC tiene `motor: 'dinamica'` y abre el panel. Las demás muestran su badge "Motor Legacy" y, si se hace click, redirigen al sistema antiguo (`costostransportetienda`).
2. **Panel de configuración** — header fijo con el nombre de la transportadora y su motor real (leído de `ConfiguracionLiquidacionTransportadora`, no del dato fijo de la card), más 5 pestañas.

---

## Las 5 pestañas

| Pestaña | Qué configura |
|---|---|
| [Configuración General](components/configuracion-general.md) | Motor de liquidación, política de peso, tope de kilos, trayecto por defecto |
| [Trayectos](components/trayectos.md) | Catálogo de códigos de ruta válidos |
| [Tarifas por Tienda](components/tarifas-tienda.md) | Comisiones, seguro, flete por trayecto y tabla de incrementos, por tienda o genérica |
| [Simulador de Liquidación](components/simulador-liquidacion.md) | Formulario preparado para probar "¿cuánto liquidaría esta guía?" — todavía sin cálculo real |
| [Auditoría de Excepciones](components/auditoria-excepciones.md) | Guías que el motor Dinámico no pudo liquidar, con su motivo |

Los badges numéricos de Trayectos y Tarifas se calculan en vivo desde el store (antes eran números fijos que quedaban desactualizados apenas se agregaba o borraba algo).

---

## Servicios y endpoints

| Servicio | Método | Endpoint | Cuándo |
|---|---|---|---|
| [`TransportadoraActivaService`](servicios/transportadora-activa-service.md) | `seleccionar(card)` | `GET metodoGenerico?coleccion=ConfiguracionLiquidacionTransportadora\|TrayectosTransportadora\|TarifasTransportadoraTienda\|AuditoriaLiquidaciones\|Tienda` | Al elegir una transportadora en el Hub — las 5 consultas corren en paralelo |

Todo pasa por el mismo CRUD genérico (`metodoGenerico?coleccion=...`) que usa el resto de la plataforma — no hay un endpoint HTTP propio de este módulo.

---

## Roles y permisos

`ROLES_TRANSPORTADORAS` es un mapa simple: hoy un único grupo de roles (`Director de Operaciones`, `CEO`, `Administrador`, `COO`, `Desarrollador`) tiene acceso a **todas** las transportadoras (comodín `'*'`). Está preparado para restringir un rol a transportadoras puntuales el día que haga falta (agregar una entrada con un array de `keys` en vez de `'*'`), pero hoy nadie lo necesita.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-09-16 | Iker Acevedo | Documentación inicial del módulo completo: Hub, panel, 5 pestañas, servicio compartido. |

---

## Observaciones

- El header del panel muestra cuántas tiendas del catálogo ya tienen tarifa propia configurada (vs. dependen de la Genérica) — de un vistazo, sin entrar a la pestaña de Tarifas.
- Ver [ADR-001](adr/ADR-001-simulador-sin-calculo-propio.md) para por qué el Simulador no calcula nada todavía, y por qué esa fue una decisión deliberada, no una funcionalidad a medias.
