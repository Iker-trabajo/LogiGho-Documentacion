## Autor: Iker Acevedo
Fecha creacion: 2026-09-16
Estado: aceptada

# ADR — Strangler Fig para el motor de liquidación, en vez de reescribir el Legacy

**Autor:** Iker Acevedo
**Fecha:** 2026-09-16
**Estado:** Aceptada

---

## Contexto

`DepurarLiquidacionUseCase.DepuraLiquidacionInter` es el proceso que calcula cuánto le queda a cada tienda por cada guía — uno de los procesos más sensibles de la empresa, porque mueve plata real todos los días. Con el tiempo se volvió un método de ~350 líneas con toda la lógica de negocio mezclada: clasificación, búsqueda de tarifa con fallbacks en cascada, cálculo de flete por switch de nombre de trayecto, construcción de hasta 4 documentos distintos por guía, y pagos de Marketing/Referido como side-effect dentro del mismo ciclo. Agregar una transportadora nueva, o cambiar una regla de una existente, significa tocar ese método y desplegar — sin una forma de probarlo contra pedidos reales sin arriesgar la liquidación de producción.

Se necesitaba un motor nuevo, con tarifas y reglas configurables desde el front (sin desplegar código), dominio limpio y testeable. La pregunta era cómo llegar ahí sin poner en riesgo un proceso que ya funciona y que nadie puede darse el lujo de que falle un día.

---

## Opciones consideradas

### Opción A — Reescribir `DepurarLiquidacionUseCase` completo, de una vez

Reemplazar el método actual por el diseño nuevo en un solo cambio, para todas las transportadoras a la vez.

**Pros:** un solo motor, sin código duplicado, migración más corta en el calendario.
**Contras:** cualquier diferencia de comportamiento —por mínima que sea— se nota en la próxima liquidación real, para **todas** las transportadoras al mismo tiempo. Sin forma de comparar contra el comportamiento anterior antes de que la plata ya se haya movido.

### Opción B — Strangler Fig: motor nuevo al lado del viejo, transportadora por transportadora

Construir el dominio nuevo aparte (`LiquidacionDinamica/`), sin tocar el Legacy, y decidir por configuración (`ConfiguracionLiquidacionTransportadora.MotorLiquidacion`) qué transportadora usa cuál motor. Empezar por una sola transportadora (TCC), con un modo `Sombra` que calcula sin persistir, para comparar antes de confiar.

**Pros:** el Legacy nunca se toca — cero riesgo de romper lo que ya funciona para Inter/Envía/Servientrega mientras se construye lo nuevo. Cada transportadora migra solo cuando el motor nuevo ya demostró (tests + guías reales comparadas a mano) que calcula igual. Un problema en el motor nuevo con TCC no afecta a las demás transportadoras.
**Contras:** dos motores conviven más tiempo — hay que mantener mentalmente que existen dos formas de llegar al mismo documento `Peticion`. Migración más lenta que un big-bang.

> No hubo una Opción C seria — dado que el proceso mueve plata real, cualquier variante de "reescribir todo de una vez" comparte el mismo riesgo de la Opción A.

---

## Decisión

**Se eligió la Opción B — Strangler Fig.**

**Razón:** el costo de una migración más lenta es aceptable; el costo de una liquidación mal calculada en un proceso financiero core no lo es. Empezar por TCC (una transportadora, en preprod, con la opción de modo `Sombra` antes de pasar a `Dinamica`) da una forma real de validar el motor nuevo contra datos de verdad sin apostar el proceso completo.

---

## Consecuencias

**Positivas:** el Legacy queda intacto y sigue liquidando con el comportamiento probado de siempre mientras el motor nuevo madura. Cada transportadora se puede migrar, y si algo sale mal, revertir (`MotorLiquidacion = Legacy`) es cambiar un campo en Mongo, no un despliegue. El dominio nuevo queda con >100 tests unitarios cubriendo cada regla por separado, algo que el método monolítico del Legacy nunca tuvo.

**Negativas:** hasta que la última transportadora migre, hay dos motores que mantener mentalmente, y cualquier cambio de "reglas de negocio compartidas" (por ejemplo, el 0.4% de Impuesto Gobierno) hay que replicarlo en los dos lugares si aplica a ambos. Los dos motores escriben en la misma colección (`LiquidacionesLogigho`) con el mismo formato — eso evita fragmentar reportes/contabilidad, pero significa que un bug de mapeo en cualquiera de los dos puede ensuciar la misma colección que usa el otro.

---

## Impacto en el código

| Módulo / Repo | Cambio |
| ---------------- | ---------------------- |
| `ApiLambdaLiquidacionesLogighoAOT` | Nueva carpeta `LiquidacionDinamica/` (dominio completo) + `LiquidarPedidosDinamicoUseCase` como orquestador de partición. `DepurarLiquidacionUseCase.cs` sin cambios. |
| `SitioLogiGho` (Angular) | Nuevo módulo `director-de-operaciones/costos-transportadora` — configuración de tarifas/trayectos/política de peso por transportadora. |
| `.preprod-hub` | Nuevo dominio AWS `financiero`: Lambda + Step Function `Financiero-PreProd`, para poder disparar liquidaciones de prueba en preprod sin tocar producción. |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-09-16 | Iker Acevedo | Creación del ADR, documentando una decisión ya tomada y en ejecución (TCC en preprod). |

---

## Referencias

- [Motor Legacy](../legacy.md)
- [Motor Dinámico — Visión general](../motor-dinamico/overview.md)
- [Frontend — Costos Transportadora](../../../../../frontend/views/director-de-operaciones/costos-transportadora/costos-transportadora.md)
