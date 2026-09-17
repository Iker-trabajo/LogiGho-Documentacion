## Autor: Iker Acevedo
Fecha creación: 2026-09-16  
Estado: producción (Motor Legacy) + preprod (Motor Dinámico)

# Liquidaciones — Visión general

**Liquidar** una guía es calcularle a la tienda cuánto le queda de esa venta después de descontar flete, servicio, seguro, comisión de recaudo, impuesto de gobierno, y —si hay un proveedor dropshipper detrás— su parte también. Es uno de los procesos más sensibles de LogiGho porque mueve dinero real: si el cálculo está mal, alguien cobra de más o de menos, y eso se nota de inmediato.

Hoy conviven **dos motores** dentro de la misma Lambda, `ApiLambdaLiquidacionesLogighoAOT`:

- **Motor Legacy** — calcula liquidaciones desde siempre. Un único método de ~350 líneas (`DepurarLiquidacionUseCase.DepuraLiquidacionInter`) con toda la lógica hardcodeada: switches por nombre de trayecto, fallbacks en cascada, reglas de tarifa entremezcladas con aritmética. Funciona y está en producción. Liquida hoy Interrapidísimo, Envía y Servientrega.
- **Motor Dinámico** — arquitectura nueva, dominio limpio desde cero (modelos, calculadoras, políticas, validadores — cada uno con su propio test). En Legacy, las reglas de una transportadora viven en código; aquí viven en Mongo: tarifas, trayectos y política de peso son **configuración**, editable desde el módulo *Costos Transportadora* del front. Hoy liquida TCC. El diseño permite migrar cualquier transportadora solo con crear su configuración, sin tocar código.

Hoy coexisten. La Lambda decide, pedido por pedido, cuál motor usa cada uno.

---

## Por qué dos motores en vez de reescribir Legacy

Reescribir un proceso que mueve dinero real de una día para otro es arriesgado: si falla, falla para todos a la vez. Se aplicó el patrón **Strangler Fig**: motor nuevo crece junto al viejo, transportadora por transportadora. El viejo se desactiva solo cuando el nuevo demuestra que calcula igual (o mejor, documentando cada diferencia). Ver [ADR-001](adr/ADR-001-strangler-fig-motor-dinamico.md).

**Consecuencia clave:**
- **Legacy nunca se modificó.** `DepurarLiquidacionUseCase.cs` es el archivo original.
- Código nuevo vive en carpeta aparte (`LiquidacionDinamica/`).
- Ambos motores escriben en **la misma colección** Mongo (`LiquidacionesLogigho`), mismo formato.
- Para contabilidad y reportes: no hay diferencia visible entre liquidación Legacy o Motor Dinámico.

---

## Cómo se decide qué motor usa cada pedido

Colección `ConfiguracionLiquidacionTransportadora` (un documento/transportadora) define qué motor usa. Campo `MotorLiquidacion`:

| Valor | Comportamiento |
|---|---|
| `Dinamica` | Motor Dinámico calcula y persiste resultado. |
| `Sombra` | Motor Dinámico calcula pero **no guarda** — valida configuración sin arriesgar dinero real. |
| `Legacy` | Motor Legacy procesa como siempre (Motor Dinámico ignorado). |
| `Deshabilitado` | Transportadora no liquida — ambos motores la saltan. |
| *(sin config)* | Mismo que `Legacy` — comportamiento seguro por defecto. Sin configuración explícita, transportadora sigue ruta original. |

`LiquidarPedidosDinamicoUseCase.Particionar()` hace la separación:
1. Agrupa configuraciones por `NombreEnPedidos` (normalizado).
2. Compara campo `Transportadora` de cada pedido contra ese diccionario.
3. Genera dos lotes: `Dinamicos` y `Legacy`.
4. Ambos se procesan **en misma ejecución Lambda** — no son mutuamente excluyentes.

Detalles en [Orquestador](motor-dinamico/orquestador.md).

---

## Diagrama de flujo (estilo BPMN)

Vista de punto a punto: desde que llega la orden de liquidar hasta que el documento queda en Mongo, con las dos "carriles" (pools) — Motor Legacy y Motor Dinámico — corriendo en la misma invocación.

```mermaid
--8<-- "backend/lambdas-dotnet/lambdas/ApiLambdaLiquidacionesLogighoAOT/diagramas/overview-bpmn.mmd"
```

> Fuente y demás diagramas de este módulo: **[carpeta de diagramas](diagramas/README.md)**.

---

## Mapa de la documentación

| Página | Qué encontrás ahí |
|---|---|
| [Lambda (contrato técnico)](ApiLambdaLiquidacionesLogighoAOT.md) | Trigger, request/response, variables de entorno |
| [Motor Legacy](legacy.md) | Cómo liquida hoy `DepurarLiquidacionUseCase`, sus reglas y sus puntos frágiles |
| [Motor Dinámico → Visión general](motor-dinamico/overview.md) | Filosofía de diseño, flujo del dominio guía por guía |
| [Motor Dinámico → Orquestador](motor-dinamico/orquestador.md) | Partición Legacy/Dinámica/Sombra, `Function.cs`, fases del proyecto |
| [Motor Dinámico → Modelos de dominio](motor-dinamico/modelos.md) | Configuración, Pedido, Resultados de cálculo |
| [Motor Dinámico → Calculadoras](motor-dinamico/calculadoras.md) | Cada fórmula, una por una |
| [Motor Dinámico → Políticas de peso](motor-dinamico/politicas-peso.md) | Tabla de incrementos (TCC) vs. porcentaje sobre flete (resto) |
| [Motor Dinámico → Validación](motor-dinamico/validacion.md) | Clasificador de tipo, resolución de tarifa, validación de trayecto |
| [Motor Dinámico → Infraestructura](motor-dinamico/infraestructura.md) | Mapeo Mongo ↔ dominio, por qué escribe en la misma colección |
| [Despliegue en PreProd (Financiero)](orquestacion-financiero.md) | Step Function `Financiero-PreProd`, cómo se desplegó |
| [Operación](operacion.md) | Cómo correr una liquidación de prueba a mano, guía por guía |
| [Frontend — Costos Transportadora](../../../../frontend/views/director-de-operaciones/costos-transportadora/costos-transportadora.md) | Pantalla donde se configura el motor Dinámico |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-09-16 | Iker Acevedo | Documentación inicial completa: motor Legacy, motor Dinámico, despliegue en PreProd y frontend de configuración. |
