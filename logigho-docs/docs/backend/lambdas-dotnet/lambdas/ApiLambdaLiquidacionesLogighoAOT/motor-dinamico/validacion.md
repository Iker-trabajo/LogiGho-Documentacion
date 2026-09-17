## Autor: Iker Acevedo
Fecha creación: 2026-09-16

Estado: preprod

# Validación

Tres clases, cada una en `LiquidacionDinamica/Dominio/Servicios/`, que deciden si una guía puede avanzar por el pipeline — y si no puede, por qué exactamente. Ninguna hace I/O: reciben los datos ya cargados (pedido, configuración, tarifas, trayectos) y devuelven una decisión.

---

## `ClasificadorTipoLiquidacion`

Port de `DepurarLiquidacionUseCase.cs:76-87`: **Entrega** si hay Fecha Entrega y el estado está en la lista `EstadosEntregaLegacy` (`Entregada`, `Digitalizada`, `Archivada`, `Pagada`); **Devolución** si hay Fecha Devolución. Si ninguna de las dos aplica, devuelve `null`.

Esa última parte es la diferencia real frente a Legacy: Legacy, sin fecha de ninguno de los dos tipos, **sigue de largo y persiste una liquidación en cero** para esa guía. El motor Dinámico corta ahí — una guía sin tipo determinado no se liquida, queda en auditoría (`MotivoNoLiquidacion.TipoNoDeterminado`).

No decide qué bloques de cálculo corren después de clasificar — eso son dos ramas independientes tanto en el legacy como en el orquestador del motor Dinámico.

## `ValidadorTrayecto`

Valida el trayecto crudo del pedido contra la lista blanca de trayectos activos de la transportadora (`TrayectosTransportadora`). Si no es válido, intenta caer al `TrayectoPorDefecto` configurado en `ConfiguracionTransportadora` — y solo si ese default **también** está en la lista blanca. Si ninguno de los dos sirve, la guía va a auditoría (`MotivoNoLiquidacion.TrayectoSinResolver`).

Diferencia frente a Legacy: Legacy, ante un trayecto no reconocido, cae **en silencio** a `FleteNacionalMetropolitano` — sin dejar rastro de que pasó. `ValidadorTrayecto` lo hace explícito y auditable.

## `ResolvedorTarifa`

Busca la tarifa que aplica a un pedido, **siempre por `IdTienda`** — nunca por nombre de tienda, a propósito, para no heredar los bugs de matching por texto que arrastra el legacy (ver la cascada de `FirstOrDefault` en [Motor Legacy](../legacy.md#cómo-decide-el-flete)). Si la tienda no tiene tarifa propia para ese tipo de liquidación (o el pedido no trae tienda), cae a la tarifa Genérica de la transportadora. Si tampoco hay Genérica, la guía va a auditoría (`MotivoNoLiquidacion.TarifaNoEncontrada`).

---

## Catálogo de `MotivoNoLiquidacion` (9 motivos)

Motivos reales que produce Motor Dinámico. El front, en versión 1, inventó motivos extra (`sobrepeso_sin_regla`). Se corrigió — ahora coinciden 1:1 backend ↔ frontend.

| Motivo | Cuándo pasa | ¿Se corrige configurando algo? |
|---|---|---|
| `TrayectoSinResolver` | Trayecto no reconocido y sin default configurado | Sí — pestaña Trayectos |
| `TarifaNoEncontrada` | No hay tarifa de tienda ni genérica | Sí — pestaña Tarifas |
| `ConfiguracionTransportadoraAusente` | La transportadora no tiene ningún documento de configuración | Sí — pestaña Configuración General |
| `MotorDeshabilitado` | `MotorLiquidacion = Deshabilitado` | Sí — pestaña Configuración General |
| `TipoNoDeterminado` | Sin Fecha Entrega ni Fecha Devolución | No — problema del dato del pedido, no de configuración |
| `DatoPedidoInvalido` | El pedido trae un dato inválido o incompleto | No |
| `ErrorCalculo` | Excepción durante el cálculo | No |
| `PersistenciaFallida` | El cálculo se hizo bien pero no se pudo guardar | No |
| `AdicionalNoPersistido` | Un costo adicional (marketing/referido) no se pudo guardar | No |

Esta tabla y sus textos legibles viven espejados en el front — ver [`MOTIVO_NO_LIQUIDACION_TEXTO`](../../../../../frontend/views/director-de-operaciones/costos-transportadora/modelos/liquidacion-transportadora-models.md).

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-09-16 | Iker Acevedo | Documentación inicial de las 3 validaciones y el catálogo de motivos de no-liquidación. |
