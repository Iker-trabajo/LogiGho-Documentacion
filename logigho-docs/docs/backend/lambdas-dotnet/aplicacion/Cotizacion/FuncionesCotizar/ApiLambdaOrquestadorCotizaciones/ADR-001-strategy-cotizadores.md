---

## Autor: Iker Acevedo
Fecha creacion: 2026-09-09
Estado: aceptada

# ADR — Patrón Strategy para cotizadores + semántica de recaudo por transportadora

**Autor:** Iker Acevedo
**Fecha:** 2026-09-09
**Estado:** Aceptada

---

## Contexto

El cotizador orquesta 3 transportadoras. Antes de esta feature:

- **Envía nunca se invocaba de verdad**: el orquestador estimaba su tarifa geográfico y hacia el calculo algoritmicamente, sin tocar la API real de Envía.
- **Inter y Servientrega sí cotizaban real, pero siempre con recaudo forzado** — no existía el concepto de "sin recaudo" para ellas (Servientrega tenía el campo `EnvioConCobro` en su contrato, pero el mapeo lo forzaba a `true`).
- `DirectFunction` (el orquestador) tenía un bloque `if/else` duplicado por transportadora, de casi 200 líneas, mezclando armado de request, invocación y aplicación de la fórmula Logigho.

Al integrar Envía real y agregar el switch "con/sin recaudo" a las 3 transportadoras, agregar una lógica más al `if/else` existente iba a hacerlo insostenible. Además se necesitaba una forma de testear cada transportadora sin invocar AWS de verdad.

---

## Opciones consideradas

### Opción A — Seguir con `if/else` en `DirectFunction`

Agregar el `ConRecaudo` y la integración de Envía dentro del mismo bloque condicional.

**Pros:** cambio mínimo, no hay que tocar la estructura del proyecto.
**Contras:** el método ya era difícil de leer con 2 transportadoras, con 3 y con recaudo variable se vuelve aumentabamos la complejidad del codigo y lo haciamos insostenible. Cada transportadora nueva futura implicaría tocar el orquestador entero. Difícil de testear en aislamiento.

### Opción B — Patrón Strategy (`ICotizadorProveedor`)

Cada transportadora implementa una interfaz común (`ICotizadorProveedor`): mapea su request, invoca su lambda (con un constructor `internal` alterno para inyectar un invoker mockeado en tests) y aplica la fórmula Logigho si `CotizacionLogigho = true`. Un `RegistroCotizadores` actúa de catálogo. `DirectFunction` arma el contexto común y corre el catálogo en paralelo.

**Pros:** `DirectFunction` baja de ~200 a ~55 líneas. Agregar una transportadora nueva es 1 clase + 1 línea en el registro, sin tocar el orquestador (Open/Closed). Cada estrategia se testea en aislamiento con mocks.
**Contras:** una capa de indirección más para quien no conoce el patrón; ligero costo de aprendizaje inicial.
--

## Decisión

**Se eligió:** Opción B — patrón Strategy con `ICotizadorProveedor` + `RegistroCotizadores`.

**Razón:** el problema real no era solo "agregar Envía", era que el orquestador iba a seguir creciendo (con/sin recaudo, futuras transportadoras). Strategy separa el "qué transportadoras existen" del "cómo se orquesta la cotización", que es justo el eje que estaba cambiando.

---

## Semántica de forma de pago / recaudo por transportadora

La parte no obvia que motivó documentar esto aparte: **cada transportadora expresa "con/sin recaudo" distinto**, y Envía está **invertida** respecto a las otras dos.

| Transportadora | Con recaudo | Sin recaudo |
| --------------- | ----------- | ----------- |
| Interrapidísimo | `AplicaContrapago = true`, `ValorDeclarado` = valor real del pedido | `AplicaContrapago = false`, mismo `ValorDeclarado` (no hay piso fijo) |
| Servientrega | `EnvioConCobro = true` | `EnvioConCobro = false` |
| Envía | Cuenta `ENVIA_CUENTA_RECAUDO` → `cod_formapago = 4` (**Crédito**) | Cuenta `ENVIA_CUENTA_SIN_RECAUDO` → `cod_formapago = 7` (**Contraentrega**) |


> 💡 **Por qué importa la inversión de Envía**: si se replica ciegamente la regla de Inter/Servientrega (con recaudo → contado/contraentrega) a Envía, la cuenta y la forma de pago quedan cruzadas y Envía facturará mal. Esta es la razón por la que existe `EnviaCuentaResolver` como pieza separada en vez de una condición más dentro del mapeo genérico.

---

## Consecuencias

**Positivas:** orquestador simple y testeable; agregar transportadoras futuras no toca código existente; la regla de negocio de cada transportadora vive en un solo lugar (su propia clase `CotizadorXxx`).

**Negativas:** las 2 cuentas de Envía son globales para toda la plataforma (no por tienda) — si en el futuro se necesita una cuenta de Envía por tienda, `EnviaCuentaResolver` hay que rediseñarlo. Queda como deuda técnica que la guía con recaudo para Envía sin-recaudo no se factura bien operativamente (fuera de alcance de esta feature).

---

## Impacto en el código

| Módulo / Repo | Cambio |
| ------------- | ------ |
| `LambdasLogiGho` (`ApiLambdaOrquestadorCotizaciones`) | Nuevo `ICotizadorProveedor`, `CotizadorEnvia`/`CotizadorInter`/`CotizadorServientrega`, `RegistroCotizadores`, `EnviaCuentaResolver`. `DirectFunction` reducido a orquestar el catálogo. |
| `LambdasLogiGho` (`ApiLambdaCotizarEnvia`) | `EnviaLiquidacionRequest` completado con datos de cuenta. Nueva `EnviaLiquidacionException` para el bug de Envía respondiendo `200` con error de negocio. |
| `LambdasLogiGho` (`ApiLambdaCotizarInterrapidisimo`) | Separación de `ValorDeclarado` y `ValorContraPago` en el DTO/mapeo (antes se mandaba mal). Eliminado `IdFormaPago` (campo muerto). |
| `LambdasLogiGho` (`ApiLambdaGenerarCotizacion`) | Sin cambios en la lambda — solo se dejó de forzar `EnvioConCobro = true` en `FromCommonMapper.ToServientrega`. |
| `SitioLogiGho` | Switch "Con recaudo / Sin recaudo" ahora aplica a las 3 transportadoras. Ver `frontend/components/paso-cotizacion.md` y `frontend/core/reglas-modalidad-pago.md`. |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-09-09 | Iker Acevedo | Creación del ADR |

---

## Referencias

- [ApiLambdaOrquestadorCotizaciones](ApiLambdaOrquestadorCotizaciones.md)
- [ApiLambdaCotizarEnvia](../ApiLambdaCotizarEnvia/ApiLambdaCotizarEnvia.md)
- [ApiLambdaCotizarInterrapidisimo](../ApiLambdaCotizarInterrapidisimo/ApiLambdaCotizarInterrapidisimo.md)
- [reglas-modalidad-pago.ts (front)](../../../../../../frontend/core/reglas-modalidad-pago.md)
