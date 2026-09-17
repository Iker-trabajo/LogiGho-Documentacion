## Autor: Iker Acevedo
Fecha creación: 2026-09-16

Estado: preprod

# Políticas de Peso

Deciden el "flete configurado" de una guía **antes** de que `CalculadoraFleteTienda` lo compare contra lo que realmente cobró la transportadora. Cada transportadora usa una sola política, definida en `ConfiguracionTransportadora.PoliticaPeso`.

Interfaz común: `IPoliticaPeso.CalcularFleteConfigurado(ContextoCalculoFlete contexto)`.

---

## `PoliticaPorcentajeSobreFleteTransportadora`

Usada hoy por **Interrapidísimo, Envía, Servientrega** (D2/D2E incluidos) — es la política que corresponde al modelo mental del motor Legacy, portada como dominio puro.

Regla: si el peso es mayor a 1kg, **ignora el trayecto** y cobra un porcentaje de recargo (`PorcentajeKiloAdicional`) sobre lo que **realmente** cobró la transportadora (`TotalFlete`, el valor real de la guía, no el configurado). Si el peso es 1kg o menos, usa el flete de tarifa del trayecto tal cual, sin recargo.

---

## `PoliticaTablaIncrementosConTope`

Usada por **TCC** — es la política nueva, la que el modelo de porcentaje no podía representar bien porque TCC negocia el costo kilo a kilo, no como un recargo plano.

Regla:

1. Redondea el peso a kilos enteros.
2. Suma los incrementos de la tabla (`IncrementosPeso`) desde el flete base de 1kg hasta el peso del pedido, o hasta el tope configurado (`TopeKilosTabla`) — lo que sea menor.
3. Si la tarifa de la tienda **no trae incrementos propios** para ese trayecto, cae al genérico **solo para la tabla de incrementos** — el flete base de 1kg y los porcentajes siguen siendo los propios de la tienda. Una tienda puede negociar su flete sin tener que redefinir también toda la tabla de kilos.
4. Si el peso supera el tope, aplica el mismo `%KiloAdicional` que usa la otra política — pero sobre el flete **ya calculado en el tope**, no sobre `TotalFlete` real.

Pseudocódigo (`porcentajeKiloAdicional` vive en `TarifaTransportadoraTienda`, tope en `ConfiguracionTransportadora.TopeKilosTabla`):

```csharp
// Suma base (1kg) + incrementos hasta tope
int kilosParaTabla = Math.Min(kilos, tope);
double fleteConfigurado = flete1kg;  // flete base
for (int kilo = 2; kilo <= kilosParaTabla; kilo++)
    fleteConfigurado += incrementosTabla[kilo] ?? 0;

// Si supera tope, recargo porcentual
if (kilos > tope)
    fleteConfigurado *= (1 + porcentajeKiloAdicional / 100);

return fleteConfigurado;
```

---

## Por qué son dos clases separadas y no un `if` dentro de una sola

Cada política tiene semántica distinta de "peso":
- **Porcentaje:** recarga sobre total real (ignora trayecto detalle).
- **Tabla:** flete kilo a kilo desde trayecto.

Si estuvieran en una clase con `if(politica == ...)`, agregar tercera (ej: rango-de-peso) implicaría tocar clase sirviendo dos transportadoras reales, arriesgando ambas.

Con `IPoliticaPeso`: política nueva = clase nueva, sin tocar existentes, cada una con tests propios. **Single Responsibility + Open/Closed** (SOLID).

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-09-16 | Iker Acevedo | Documentación inicial de las 2 políticas de peso. |
