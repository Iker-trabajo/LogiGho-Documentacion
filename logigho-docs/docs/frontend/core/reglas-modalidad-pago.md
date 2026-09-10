---

## Autor: Iker Acevedo

Fecha creacion: 2026-09-09
Estado: produccion

# Módulo de dominio: reglas-modalidad-pago.ts

**Ubicación:** `src/app/core/domain/reglas-modalidad-pago.ts`
**Usado por:** paso 4 (Guía) del `ModalCreacionPedidosComponent`

---

## ¿Qué hace?

**Single source of truth** de cómo el switch "con/sin recaudo" (elegido en el paso 3, cotización) se traduce a los campos `FORMA DE PAGO` y `APLICA CONTRA PAGO` que se envían al generar la guía, por transportadora. Si vas a agregar una transportadora nueva al cotizador/guía, **esta es la tabla que hay que tocar**.

Existe porque había un bug real en producción: Envía estaba **forzado** a `FORMA DE PAGO = CREDITO` / `APLICA CONTRA PAGO = NO` sin importar el switch, porque había 2 sitios calculando lo mismo (`prepararDatosParaEnvio` y un bloque duplicado dentro de `generarGuia()`, este último con la etiqueta vieja `"CONTRA PAGO"` que además excluía a Envía de la lógica). Se centralizó todo acá y se borró el duplicado.

---

## Regla de negocio

`APLICA CONTRA PAGO` y `Total recaudo` son **universales** — siempre siguen el switch de cotización, sin importar la transportadora. Lo único que cambia entre transportadoras es la **etiqueta** de "FORMA DE PAGO":

| Transportadora | Con recaudo | Sin recaudo |
| --------------- | ----------- | ----------- |
| Interrapidísimo | `FORMA DE PAGO = CONTADO`, `APLICA CONTRA PAGO = SI` | `FORMA DE PAGO = CREDITO`, `APLICA CONTRA PAGO = NO` |
| Servientrega | `FORMA DE PAGO = CONTADO`, `APLICA CONTRA PAGO = SI` | `FORMA DE PAGO = CREDITO`, `APLICA CONTRA PAGO = NO` |
| Envía | **`FORMA DE PAGO = CREDITO`**, `APLICA CONTRA PAGO = SI` | **`FORMA DE PAGO = CONTADO`**, `APLICA CONTRA PAGO = NO` |

⚠️ **Envía está invertida** respecto a Inter y Servientrega — mismo patrón de inversión que ya existe en el backend (`EnviaCuentaResolver`, ver [ADR-001](../../backend/lambdas-dotnet/lambdas/ApiLambdaOrquestadorCotizaciones/ADR-001-strategy-cotizadores.md)). Este es justo el caso que estaba mal en producción antes de esta feature.

**Si se agrega una transportadora nueva sin entrada en `REGLAS_MODALIDAD_PAGO`**: cae a la regla estándar (Contado con recaudo / Crédito sin recaudo) y **avisa por consola** — no falla en silencio.

---

## API del módulo

| Función | Descripción |
| ------- | ----------- |
| `resolverFormaPago(transportadora, conRecaudo)` | Devuelve la etiqueta de `FORMA DE PAGO` según la tabla de arriba. |
| `resolverAplicaContraPago(conRecaudo)` | Devuelve `SI`/`NO`. Universal, no depende de la transportadora. |
| `resolverTotalRecaudo(conRecaudo, valorDeclarado)` | Devuelve el total a recaudar (universal). |
| `describirRegla(transportadora)` | Texto legible de la regla, usado para mostrarla en la UI del paso 4 (para que nunca se desincronice el texto mostrado de la lógica real). |

---

## UI del paso 4 (Guía)

- Label "Tipo de Pago" → **"Modalidad de pago"** (mismo lenguaje que la cotización del paso 3).
- Advertencia visible: la modalidad de pago de la guía puede diferir de lo elegido en la cotización si el usuario cambia el switch entre pasos.
- Reglas visibles debajo del switch, leídas de `describirRegla()` — nunca se desincroniza el texto mostrado de la lógica real.
- Botones a 50%/50% de ancho, igual que los banners del resto del modal.

---

## Endpoints que consume

Este módulo no consume endpoints. Es lógica pura de dominio (funciones sin efectos secundarios).

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-09-09 | Iker Acevedo | Creación del módulo. Corrige el bug de Envía forzado a Crédito/No-contra-pago. Se eliminó el bloque duplicado dentro de `generarGuia()`. |

---

## Observaciones

- Tests: `reglas-modalidad-pago.spec.ts` (17 tests) — cubre especialmente el caso Envía invertido, que es el que estaba mal en producción.
- Antes de este módulo, el paso 4 tenía **2 lugares distintos calculando forma de pago** (`prepararDatosParaEnvio` y un bloque suelto en `generarGuia()`). Si en el futuro aparece un tercer sitio calculando esto, es una señal de que se está repitiendo el bug — todo debería pasar por acá.
