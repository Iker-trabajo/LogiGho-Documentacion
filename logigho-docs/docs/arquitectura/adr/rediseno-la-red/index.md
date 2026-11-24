# Decisiones del rediseño «La Red» (ADR)

Decisiones de fondo del [rediseño «La Red»](../../../frontend/tema-la-red.md) del front, en el formato de la [plantilla ADR](../../../plantillas/PLANTILLA-ADR.md). Las decisiones finas (D1 a D60) viven en `orquestacion.md` de la carpeta `specs/021_rediseno_visual/` del workspace; aquí están las que definen el diseño.

!!! warning "Estado a 2026-11-23: verificado en local, pendiente de PreProd"
    Las decisiones están **implementadas y verificadas en local**; ninguna está en PreProd ni en producción. Solo «noche por defecto» tiene una aprobación de Iker registrada (2026-11-22). Las demás se marcan como «implementada, pendiente de aval» o «abierta».

| ADR | Decisión | Estado | Decisiones de origen |
|---|---|---|---|
| [ADR-RED-001](ADR-RED-001-tema-por-tokens-y-atributo-raiz.md) | El tema es una capa de tokens `--lg-*` activada por `data-lg-tema` en `<html>` | Aceptada (dirección aprobada por Iker 2026-11-22) | S1, D5, D13, D17 |
| [ADR-RED-002](ADR-RED-002-noche-por-defecto-y-claro-por-atributo.md) | Noche por defecto; claro por atributo y sin seguir la preferencia del sistema | Aceptada (noche); claro implementado, pendiente de aval; `prefers-color-scheme` abierta | S2, S4, D10, D12 |
| [ADR-RED-003](ADR-RED-003-typescript-nuevo-solo-en-el-armazon.md) | TypeScript nuevo solo en el armazón; la lógica de las vistas no se toca (con una excepción pedida por Iker) | Aceptada, con excepción declarada | §5.1, D21, D35 |
| [ADR-RED-004](ADR-RED-004-sin-view-transitions.md) | No usar View Transitions; transición entre módulos con atributos y CSS | Implementada, pendiente de aval | D42, D43 |
| [ADR-RED-005](ADR-RED-005-precarga-selectiva.md) | Precarga selectiva en lugar de `PreloadAllModules` | Implementada; coste en teléfono abierto | D42, D57 |
| [ADR-RED-006](ADR-RED-006-modo-basico.md) | Modo básico por detección temprana y sonda en ejecución | Implementada; umbral de 30 fps abierto | D51, D54 |

## Qué decisiones esperan a Iker

Resumidas de `ENTREGA.md` §7; la tabla completa está en [Rediseño «La Red»](../../../frontend/tema-la-red.md#decisiones-abiertas-de-iker): seguir o no `prefers-color-scheme`, el umbral de 30 cuadros por segundo del modo básico, el coste de +1 % de carga y +33 a +82 ms en teléfono lento, el muro, la línea de progreso, el login claro, `registercontact`, SweetAlert y la landing.
