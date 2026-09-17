# Diagramas — Liquidaciones

Fuente de todos los diagramas Mermaid usados en la documentación de `ApiLambdaLiquidacionesLogighoAOT`. Se mantienen como archivos `.mmd` separados (en vez de embebidos en cada `.md`) para poder reutilizarlos entre páginas sin duplicar código — ver `pymdownx.snippets` en `mkdocs.yml`.

| Archivo | Se usa en | Qué muestra |
|---|---|---|
| `overview-bpmn.mmd` | [Visión general](../overview.md) | Flujo completo estilo BPMN: partición Legacy/Dinámica/Sombra, ambos carriles hasta la persistencia en `LiquidacionesLogigho` |
| `flujo-legacy.mmd` | [Motor Legacy](../legacy.md) | Flujo interno de `DepurarLiquidacionUseCase.DepuraLiquidacionInter` |
| `flujo-motor-dinamico.mmd` | [Motor Dinámico → Visión general](../motor-dinamico/overview.md) | Flujo del dominio guía por guía: `PedidoLiquidable` → Clasificador → Validador → Resolvedor → Calculadoras → `ConstructorPeticion` |
| `orquestador-particion.mmd` | [Motor Dinámico → Orquestador](../motor-dinamico/orquestador.md) | Cómo `Particionar()` decide, transportadora por transportadora, qué lote recibe cada pedido |

> Nota sobre notación: Mermaid no tiene un modo BPMN 2.0 nativo (piscinas/carriles reales, eventos de mensaje, etc.). `overview-bpmn.mmd` usa un `flowchart` con la simbología equivalente (óvalos para eventos de inicio/fin, rectángulos para tareas, rombos para compuertas, `subgraph` como carril) para que se lea con la misma lógica que un BPMN sin depender de una librería externa que este sitio no tiene instalada.
