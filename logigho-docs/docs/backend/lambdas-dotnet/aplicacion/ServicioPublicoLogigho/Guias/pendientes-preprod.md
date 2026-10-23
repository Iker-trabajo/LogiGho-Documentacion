---
autor: Claude (chat de desarrollo) para Iker Acevedo
fecha_creacion: 2026-10-23
estado: abierto — Iker decide al terminar el desarrollo; ningún punto cambia código todavía
---

# Decisiones pendientes antes de PreProd

Lista de comportamientos que el código **tiene hoy** y que Iker revisará cuando termine el desarrollo. Pedido de Iker: **seguir con el plan y no
modificar estos comportamientos todavía**. Cada punto dice qué hace el código, la evidencia, las opciones y una recomendación. Los puntos P-01 a P-05 y
P-10 nacen de replicar el flujo actual de las Lambdas de creación; P-06 a P-09 son huecos operativos de la API nueva.

| Id | Tema | Decide | Estado |
| --- | --- | --- | --- |
| P-01 | `Total recaudo` igual al valor declarado sin contraentrega | Iker | Abierto |
| P-02 | `Origen = "API"` | Iker | Abierto |
| P-03 | `IdCarga = "-1"` | Iker | Abierto |
| P-04 | `ASESOR = "Logigho"` fijo | Iker | Abierto |
| P-05 | `Cantidad Total Productos` sin tope | Iker (con `reglas-negocio.md`) | Abierto |
| P-06 | `Reservada` que se reencola sin fin | Iker | Abierto |
| P-07 | `SetAsync` falla si el documento no existe → DLQ | Iker | Abierto |
| P-08 | Cupo en `OBSERVAR` | Iker | Abierto |
| P-09 | WAF sobre todo el stage de `ApiIntegracionesExternas` | Iker | Abierto |
| P-10 | Inter responde 200 sin `numeroPreenvio` en errores de validación | Iker | Abierto |
| P-11 | Decisiones de PreProd (REST nuevo, pools propios, MFA opcional, F1b en espera) | Iker | Resuelto |

---

## P-01 · `Total recaudo` = valor declarado aunque no haya contraentrega

- **Qué hace el código:** `DocumentosPedido.cs` (~línea 100) deja `Total recaudo` igual al valor declarado en todo pedido, con o sin contraentrega
  (`"Quirk actual de Inter"`, heredado de `ApiLambdaCrearGuia`).
- **Evidencia (chat principal):** la liquidación legacy pone recaudo 0 si `AplicaContraPago != "SI"`
  (`ApiLambdaLiquidacionesLogighoAOT/.../DepurarLiquidacionUseCase.cs:118`), pero para guías **sin** contraentrega la base del seguro usa el `TotalRecaudo`
  guardado (líneas 117 y 136 en adelante). Es decir: el valor guardado **sí cambia lo que se liquida**.
- **Pendiente de verificar:** el motor dinámico (`CalculadoraBaseRecaudo.cs`).
- **Opciones:** (a) **conservar** el quirk y que la calculadora de cupo lea exactamente lo mismo que la liquidación; (b) guardar 0 sin contraentrega, lo que
  cambia el seguro de las guías de la API respecto de las demás.
- **Recomendación:** (a), por paridad con el resto de guías. Y alinear [`cupo.md`](cupo.md), que hoy **supone recaudo 0**: la estimación de cupo quedaría baja
  si el seguro se calcula sobre el valor declarado.

## P-02 · `Origen = "API"`

- **Qué hace el código:** los pedidos creados por la API llevan `Origen: "API"` (`DocumentosPedido.cs`, constante `OrigenApi`).
- **Evidencia:** solo `ApiLambdaCrearGuia` distingue `"CRM"`, y el Angular `exportacion-pedidos.component.ts:184` cambia de colección si es CRM. Ningún
  proceso filtra por `"API"`.
- **Riesgo:** bajo. Un valor nuevo no activa ninguna rama existente.
- **Opciones:** (a) dejar `"API"`; (b) usar el valor de las guías manuales.
- **Recomendación:** (a); permite reportar qué viene de la API. Confirmar con Iker.

## P-03 · `IdCarga = "-1"`

- **Qué hace el código:** igual que la guía manual, `IdCarga` vale `"-1"` (`IdCargaIndividual`).
- **Pregunta:** ¿las guías de la API deben tener otro `IdCarga` (por ejemplo para reportes o para agrupar por cliente)?
- **Opciones:** (a) conservar `"-1"`; (b) un identificador por lote/cliente (cambia el documento y quizá reportes que agrupan por `IdCarga`).
- **Recomendación:** (a) hasta que haya un reporte que lo necesite.

## P-04 · `ASESOR = "Logigho"` fijo

- **Qué hace el código:** todas las guías de la API quedan con el asesor `"Logigho"` (`DocumentosPedido.cs`, constante `Asesor`).
- **Pregunta:** ¿se debe guardar el cliente API? Hacerlo **cambia el documento** y puede afectar pantallas que muestran o filtran por asesor.
- **Opciones:** (a) fijo; (b) nombre del cliente API; (c) campo nuevo aparte (`ClienteApi`) sin tocar `ASESOR`.
- **Recomendación:** (c) si hace falta trazabilidad: no altera lo que ya lee el sistema. Hoy la solicitud (`ApiSolicitudesGuia`) ya guarda el `ClienteId`.

## P-05 · `Cantidad Total Productos` sin tope

- **Qué hace el código:** suma todas las cantidades sin tope.
- **Evidencia:** `CrearGuia` suma hasta 12 y `CargarGuias` hasta 18.
- **Se define en F1b** con `reglas-negocio.md` (que aún no existe).
- **Opciones:** tope de 12, de 18, o configurable por transportadora.
- **Recomendación:** decidir con las reglas; hasta entonces no se puede cerrar la validación de `POST /v1/guias`.

## P-06 · `Reservada` que se reencola indefinidamente

- **Qué pasa:** una solicitud que agota sus entregas por `NoEnviada` vuelve a `Reservada`. El reconciliador (`Reconciliador.cs`,
  `EdadReservadaReencolar` = 2 min) la **vuelve a encolar cada vez que pasan más de 2 minutos**, y el **cupo queda reservado** todo ese tiempo.
- **Verificado en el código:** no hay tope de intentos ni de antigüedad que la pase a `Fallida` y libere el cupo, ni alarma específica.
- **Riesgo:** un fallo permanente de la transportadora retiene cupo de un cliente sin avisar a nadie.
- **Opciones:** (a) tope de intentos; (b) tope de antigüedad (por ejemplo 24 h); (c) ambos, con alarma por solicitud que llegue al tope.
- **Recomendación:** (c), pasando a `Fallida` con liberación de cupo y alarma.

## P-07 · `SetAsync` lanza si el documento no existe → la solicitud termina en la DLQ

- **Qué hace el código:** `RepositorioPedidosMongo.SetAsync` hace `UpdateOne` y lanza si no encuentra el documento (para no dar por completado un paso que
  no ocurrió). El mensaje se reintenta y, agotados los intentos, va a la DLQ.
- **Verificado:** existe la alarma `ApiGuias-<entorno>-dlq` (cualquier mensaje en la DLQ), pero **no** una específica para este caso ni un procedimiento
  escrito (runbook) de qué hacer.
- **Opciones:** (a) dejarlo así y escribir el runbook; (b) alarma/métrica propia para "documento ausente"; (c) ambos.
- **Recomendación:** (c): un documento ausente indica borrado manual o carrera con otro proceso y conviene verlo con nombre propio.

## P-08 · Cupo en `OBSERVAR`

- **Qué hace el código:** hasta que existan la **calculadora de cupo** y el **job de saldo**, la aplicación del cupo está en `OBSERVAR` (calcula y registra, no
  rechaza; ver [`cupo.md`](cupo.md)). Hoy el sustituto de la calculadora falla cerrado.
- **Requisito para salir a clientes externos:** `APLICAR`.
- **Opciones:** construir calculadora y job antes de PreProd con clientes reales, o limitar PreProd a pruebas internas en `OBSERVAR`.
- **Recomendación:** lo segundo para las primeras pruebas; ningún cliente externo sin `APLICAR`.

## P-09 · WAF asociado a todo el stage de `ApiIntegracionesExternas`

> **Actualización 2026-10-23:** en **PreProd** deja de aplicar: el REST es nuevo (`ApiIntegracionesExternas-PreProd`, ver [ADR-006](adr-006-mcp.md)) y no tiene `/pedidos` ni `/token`; el WAF se puede activar sin riesgo para esos usuarios (`crear_waf`, hoy `false`). **Sigue vigente para producción**, donde el stage `rl7zt6zak5` sí lo comparten.

- **Qué pasa:** la asociación del WAF es **a nivel de stage**, así que afecta también a `/pedidos` y `/token` (hoy 3 usuarios). Reglas administradas y límite
  por IP podrían bloquear tráfico legítimo de esos usuarios.
- **Estado en Terraform:** `crear_waf` existe en el módulo `guias-api`; no se ha aplicado nada contra cuentas reales.
- **Opciones:** (a) revisar límites contra el tráfico real de esos 3 usuarios antes de asociar; (b) arrancar el WAF en modo solo contar y pasar a bloquear
  después; (c) WAF solo sobre un stage/REST nuevo (rompe la regla de un solo REST).
- **Recomendación:** (a) + (b).

## P-11 · Decisiones de Iker para PreProd (registro, ya resueltas)

Se anotan aquí para que la lista refleje el estado de PreProd:

- REST público: **uno nuevo creado con Terraform**, regional, espejo de `rl7zt6zak5`.
- Cognito: **pools propios de PreProd** (integradores M2M y personas del MCP).
- MFA de las personas del MCP: **opcional**.
- F1b: no se empieza hasta que exista `reglas-negocio.md`.

Pendiente de Iker que nace de esto: dar valores y altas para encender el MCP (ver la lista de [ADR-006](adr-006-mcp.md#encender-el-mcp-en-preprod-lista-para-iker)).

## P-10 · Inter responde 200 sin `numeroPreenvio` ante errores de validación

- **Qué hace el código:** una respuesta 200 sin `numeroPreenvio` no se toma como éxito ni como rechazo claro: la solicitud pasa a `Incierta` y queda para
  revisión manual (Inter no tiene consulta por referencia; ver bitácora F1a).
- **Pendiente:** confirmar con **respuestas reales** de Inter qué errores de validación llegan así, para convertir los que se puedan en rechazo explícito
  (`TRANSPORTADORA_RECHAZO`) en vez de `Incierta`.
- **Opciones:** (a) dejar `Incierta` hasta tener ejemplos; (b) mapear los mensajes conocidos a rechazo.
- **Recomendación:** (a) ahora, (b) cuando haya muestras de PreProd.
