## Autor: Iker Acevedo
Fecha creación: 2026-09-16

Estado: preprod

# Calculadoras

Aritmética pura, sin I/O — cada una es un port literal de una parte puntual del método gigante de `DepurarLiquidacionUseCase`, con su propio test unitario. Viven en `LiquidacionDinamica/Dominio/Servicios/Calculadoras/`.

---

## `CalculadoraBaseRecaudo`

Port literal de `DepurarLiquidacionUseCase.cs:116-139`. Calcula:

- `TotalRecaudo` — según si la guía tiene contrapago o no.
- `TotalTransportadora` — el cobro inicial (negativo) de la transportadora.
- `TotalServicio` / `TotalSobreflete` — ya parametrizados a la tienda, con una regla especial: **si no hay contrapago, el sobreflete se calcula sobre el valor declarado, no sobre `TotalRecaudo`**.

Produce `ResultadoBaseRecaudo`.

## `CalculadoraFleteTienda`

Port de líneas 199-203. Aplica la regla que sostiene todo el negocio de este cálculo: **nunca cobrarle a la tienda menos de lo que la transportadora le cobró de verdad a LogiGho por esa guía puntual**. Si `AplicaMaximoServicioSeguro` está activo (hoy solo TCC), esa misma regla también protege comisión de recaudo y seguro, no solo el flete. Además aplica el piso fijo `MontoMinimoServicio`, que es independiente de esa regla (puede activarse aunque la transportadora haya cobrado menos que el mínimo).

Produce `ResultadoFleteTienda` (`Flete`, `Seguro`, `ComisionRecaudo`, `TotalFleteTienda`).

## `CalculadoraEntrega`

Descuentos/reconocimientos tienda en **Entrega**: flete entregado, flete final (piso: **-200** sobre `TotalTransportadora`), confirmación, adelanto, otros servicios.

⚠️ **Nota:** constante `-200` copiada de Legacy (origen no documentado; asumir comportamiento histórico). Recibe monto Marketing ya resuelto pero no decide si corresponde pagarlo — esa decisión es de `CalculadoraMarketing`.

## `CalculadoraDevolucion`

Versión simplificada de `CalculadoraEntrega` para guías **Devolución**: solo flete de devolución y otros servicios de devolución. No hay confirmación, adelanto, marketing, referido ni dropshipper — conceptualmente, una guía devuelta no generó venta, así que ninguno de esos conceptos aplica.

## `CalculadoraDropshipper`

Dos responsabilidades separadas:

- `SumarValorDropshipper` — cantidad × precio proveedor de cada producto (usa `Productos.precioproveedor`, **no** el precio de venta al público de la tienda — confirmado durante la auditoría de guías reales de este módulo).
- `Calcular` — adelanto del proveedor con **su propio** porcentaje configurado, y `Liquidacion = valor − adelanto − marketing del proveedor`.

## `CalculadoraImpuestoGobierno`

Port literal de líneas 379/388: **0.4%** del recaudo, registrado como un documento negativo aparte — nunca mezclado dentro de la liquidación de Entrega. Solo aplica en el bloque de Entrega, nunca en Devolución.

## `CalculadoraMarketing` y `CalculadoraReferido`

Deciden **si corresponde pagar**, no ejecutan el pago — a diferencia del legacy, donde este mismo cálculo dispara una escritura a Mongo en el mismo momento (ver [Motor Legacy](../legacy.md#marketing-y-referido-se-pagan-en-el-mismo-ciclo-de-cálculo)).

- `CalculadoraMarketing` — paga si el monto configurado es distinto de cero y la fecha de entrega es posterior o igual a la fecha de creación del Marketing, descontando el fee correspondiente.
- `CalculadoraReferido` — misma lógica, pero sin descontar fee, y con un control extra (`guiaYaReferida`) para no pagar la misma comisión dos veces.

---

## Por qué se separaron en 8 clases y no una sola "Calculadora"

Cada una de estas fórmulas puede cambiar por una razón de negocio completamente distinta a las demás — el 0.4% de impuesto no tiene nada que ver con la regla de "nunca menos que lo que cobró la transportadora", y ambas son independientes de si hay o no un proveedor dropshipper detrás. Juntarlas en una sola clase multiplicaría el riesgo de que tocar una fórmula rompa otra sin que ningún test lo note a tiempo (justo lo que le pasaba al método único del legacy). Separadas, cada una tiene su propio archivo de test, y un cambio en una fórmula solo puede romper su propio test — no los de las otras siete.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-09-16 | Iker Acevedo | Documentación inicial de las 8 calculadoras. |
