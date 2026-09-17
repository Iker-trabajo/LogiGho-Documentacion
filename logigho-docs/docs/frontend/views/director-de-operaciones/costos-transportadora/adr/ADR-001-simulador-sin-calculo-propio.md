## Autor: Iker Acevedo
Fecha creacion: 2026-09-16
Estado: aceptada

# ADR — El Simulador de Liquidación no reproduce el cálculo en el front

**Autor:** Iker Acevedo
**Fecha:** 2026-09-16
**Estado:** Aceptada

---

## Contexto

El módulo Costos Transportadora incluye una pestaña "Simulador de Liquidación", pensada para responder "si mando esta guía con estos datos, ¿cuánto debería liquidar?" sin esperar a que llegue una guía real. El cálculo real vive en el dominio del backend (`LiquidacionDinamica/Dominio/Servicios/Calculadoras`), y ese dominio hoy solo se invoca desde dentro del proceso de liquidación real de la Lambda — no hay un endpoint HTTP que lo exponga de forma aislada.

---

## Opciones consideradas

### Opción A — Reproducir la fórmula en TypeScript

Portar la lógica de las Calculadoras y Políticas de Peso al front, para que el Simulador calcule localmente con los datos ya cargados (tarifa, trayecto, peso).

**Pros:** el simulador funciona de inmediato, sin depender de un endpoint nuevo en el backend.
**Contras:** crea **dos implementaciones de la misma fórmula financiera**. Si una calculadora cambia en el backend (una fórmula, un piso, un porcentaje) y alguien olvida replicar el cambio en el front, el simulador empieza a dar resultados que ya no corresponden a lo que la Lambda liquidaría de verdad — y lo haría con la misma confianza visual que si estuviera bien. En un proceso que existe justamente para dar certeza sobre plata real, ese riesgo es peor que no tener simulador.

### Opción B — Dejar el formulario listo, sin cálculo, hasta que exista un endpoint de dominio

Construir toda la UI (selección de tienda, trayecto, tipo, peso) contra datos reales, pero que el botón "Simular" explique por qué todavía no calcula, en vez de mostrar un número.

**Pros:** cero riesgo de desincronización — hay una sola fuente de verdad para la aritmética, el dominio del backend. El formulario ya queda listo para conectar el día que exista el endpoint.
**Contras:** el simulador no es funcional todavía; un usuario que no lea el aviso puede pensar que es un bug.

---

## Decisión

**Se eligió la Opción B.**

**Razón:** un simulador que a veces dice la verdad y a veces no (porque una fórmula se desincronizó) es más peligroso que no tener simulador — genera confianza donde no debería haberla. Se prioriza la integridad del cálculo financiero por encima de tener la pantalla 100% funcional de entrada.

---

## Consecuencias

**Positivas:** una sola fuente de verdad para toda fórmula de liquidación (el dominio del backend). El formulario del Simulador no necesita rediseñarse cuando el cálculo se habilite — solo conectar el botón a un endpoint real.

**Negativas:** el Simulador no aporta valor funcional hasta que exista ese endpoint. Mientras tanto, cualquier persona que quiera validar una tarifa nueva tiene que hacerlo con una guía real (o el modo `Sombra` del motor, que calcula sin persistir sobre pedidos reales).

---

## Impacto en el código

| Módulo / Repo | Cambio |
| ---------------- | ---------------------- |
| `SitioLogiGho` | `SimuladorLiquidacionComponent.simular()` muestra un mensaje explicativo (`Swal.fire`) en vez de calcular. No hay interfaz `ResultadoSimulacion` en `liquidacion-transportadora.models.ts` — deliberado. |
| `LambdasLogiGho.Aplicacion` | Ninguno todavía — este ADR documenta por qué el front NO reproduce el cálculo; no hay pendiente de exponer un endpoint de dominio aislado a corto plazo. |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-09-16 | Iker Acevedo | Creación del ADR. |

---

## Referencias

- [Simulador de Liquidación](../components/simulador-liquidacion.md)
- [Motor Dinámico — Orquestador (modo Sombra)](../../../../../backend/lambdas-dotnet/lambdas/ApiLambdaLiquidacionesLogighoAOT/motor-dinamico/orquestador.md)
