## Autor: Iker Acevedo
Fecha creación: 2026-09-16
Estado: preprod
Tipo: componente

# Simulador de Liquidación

**Selector:** `app-simulador-liquidacion`
**Ubicación:** `src/app/views/director-de-operaciones/costos-transportadora/components/simulador-liquidacion`

---

## ¿Qué hace?

Formulario para simular una liquidación **sin persistir nada** — pensado para responder "si mando esta guía con estos datos, ¿cuánto debería liquidar?" antes de que llegue una guía real. Ya está conectado a datos reales: el selector de tienda usa el catálogo real, y el de trayecto usa los trayectos reales configurados para la transportadora activa.

---

## El botón "Simular" no calcula nada todavía — y es una decisión, no una funcionalidad a medias

El cálculo real del motor Dinámico corre **dentro del proceso de liquidación de la Lambda**, y ese dominio todavía no expone un endpoint HTTP que permita "probarlo" desde afuera sin que sea una guía real. Reproducir esa aritmética acá en TypeScript —aunque fuera "aproximada"— significaría tener **dos lugares calculando plata** con el riesgo real de que se desincronicen: si una fórmula cambia en el backend y alguien olvida replicarla acá, el simulador empezaría a mentir con confianza.

En vez de inventar un número, el botón explica la situación:

> "Por ahora no es posible calcular una liquidación de prueba sin que sea una guía real — el formulario ya queda listo con tienda y trayecto reales para cuando esto se habilite."

Ver el razonamiento completo en [ADR-001](../adr/ADR-001-simulador-sin-calculo-propio.md).

---

## Qué sí queda listo

- Selector de tienda (catálogo real).
- Selector de trayecto (trayectos reales de la transportadora activa, más la opción "Sin trayecto" para probar el comportamiento del default).
- Selector de tipo de liquidación (Entrega/Devolución).
- El peso siempre se redondea a kilos enteros al cambiar — no se cobra por fracciones de kilo, así que el formulario no deja simular ese caso que el motor real tampoco produce.

El día que exista un endpoint que exponga el cálculo del dominio sin persistir (el mismo camino que ya usa el modo `Sombra` internamente, ver [Orquestador](../../../../../backend/lambdas-dotnet/lambdas/ApiLambdaLiquidacionesLogighoAOT/motor-dinamico/orquestador.md)), conectar el botón es solo reemplazar el `Swal.fire` informativo por la llamada real — el formulario no necesita rediseñarse.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-09-16 | Iker Acevedo | Documentación inicial. |
