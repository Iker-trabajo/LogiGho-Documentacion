## Autor: Iker Acevedo
Fecha creación: 2026-09-16  
Estado: preprod (TCC)

# Motor Dinámico — Visión general

Reemplazo de Motor Legacy con dominio limpio: cada regla de negocio en su propia clase pequeña con su test. Nada de métodos de 300 líneas ni campos reseteados a mano. Diferencia clave del [Motor Legacy](../legacy.md):
- Legacy: clasificación + tarifa + cálculo + persistencia en un método único.
- Motor Dinámico: cada paso es una clase que hace una cosa.

**Impacto operacional:**
- **Legacy:** cambiar transportadora = cambiar código + desplegar.
- **Motor Dinámico:** tarifas, trayectos, política peso = datos Mongo, editables desde front [Costos Transportadora](../../../../../frontend/views/director-de-operaciones/costos-transportadora/costos-transportadora.md). Sin tocar código.

---

## Flujo del dominio, guía por guía

```mermaid
--8<-- "backend/lambdas-dotnet/lambdas/ApiLambdaLiquidacionesLogighoAOT/diagramas/flujo-motor-dinamico.mmd"
```

> Fuente: **[carpeta de diagramas](../diagramas/README.md)**.

Cada rombo naranja del diagrama no es un error — es una guía que **no se pudo liquidar** y queda registrada en auditoría con un motivo explícito (`MotivoNoLiquidacion`). Esa es la otra diferencia de fondo frente a Legacy: cuando algo no calza (trayecto desconocido, sin tarifa, tipo indeterminado), Legacy cae en silencio a un valor por defecto y liquida igual; el motor Dinámico **para esa guía puntual** y dice explícitamente por qué, en vez de adivinar con plata real. Ver [Validación](validacion.md).

---

## Piezas, en orden de uso

| Paso | Clase | Qué decide |
|---|---|---|
| 1 | [`ClasificadorTipoLiquidacion`](validacion.md#clasificadortipoliquidacion) | ¿Es Entrega, Devolución, o ninguna? |
| 2 | [`ValidadorTrayecto`](validacion.md#validadortrayecto) | ¿El trayecto que trae el pedido es válido para esta transportadora? |
| 3 | [`ResolvedorTarifa`](validacion.md#resolvedortarifa) | ¿Qué tarifa aplica — de la tienda, o la genérica? |
| 4 | [`CalculadoraBaseRecaudo`](calculadoras.md#calculadorabaserecaudo) | Recaudo, flete/servicio/sobreflete de la transportadora |
| 5 | [Política de Peso](politicas-peso.md) | Flete "configurado" según el peso, antes de compararlo con lo real |
| 6 | [`CalculadoraFleteTienda`](calculadoras.md#calculadorafletetienda) | Flete/seguro/comisión final que paga la tienda (nunca menos de lo que cobró la transportadora) |
| 7a | [`CalculadoraEntrega`](calculadoras.md#calculadoraentrega) + [`CalculadoraDropshipper`](calculadoras.md#calculadoradropshipper) + [`CalculadoraMarketing`/`CalculadoraReferido`](calculadoras.md#calculadoramarketing-y-calculadorareferido) | Descuentos/reconocimientos Entrega |
| 7b | [`CalculadoraDevolucion`](calculadoras.md#calculadoradevolucion) | Descuentos/reconocimientos Devolución (simple: sin confirmación, adelanto, marketing, referido, dropshipper) |
| 8 | [`CalculadoraImpuestoGobierno`](calculadoras.md#calculadoraimpuestogobierno) | 0.4% recaudo, documento aparte |
| 9 | [`ConstructorPeticion`](../legacy.md#construcción-de-documentos-4-conceptos-todo-mezclado-en-un-método) | Arma documento(s) `Peticion` final. Aquí vive fórmula `Liquidacion`. |

Quien orquesta este pipeline sobre pedidos reales (Mongo, particionamiento Legacy/Dinámica) es [`LiquidarPedidosDinamicoUseCase`](orquestador.md) — todo lo de esta tabla es dominio puro, sin I/O.

---

## Filosofía de diseño

Filosofía resumida: muchos archivos pequeños (cada uno su test) vs método gigante Legacy. La concentración en un solo lugar es lo que hace Legacy difícil de mantener (necesita releer todo el método cada cambio).

Algunas decisiones concretas que se desprenden de eso:

- **`ResolvedorTarifa` busca por `IdTienda`, nunca por nombre** — a propósito, para no heredar los bugs de matching por texto que tiene Legacy.
- **Ningún cálculo hace I/O.** Las 8 Calculadoras y las 2 Políticas de Peso son funciones puras: reciben datos ya resueltos, devuelven un resultado. Eso es lo que permite testearlas con datos de mentira sin necesitar Mongo — hoy hay más de 100 tests unitarios cubriendo solo el dominio.
- **Todo lo que no calza queda auditado, no adivinado.** Ver la tabla de `MotivoNoLiquidacion` en [Validación](validacion.md).
- **`Marketing`/`Referido` se separan en "decidir" y "ejecutar".** Las calculadoras dicen si corresponde pagar y cuánto; el pago real (I/O) queda para la capa de aplicación, todavía no completa — ver [fases del proyecto](orquestador.md#fases-del-proyecto).

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-09-16 | Iker Acevedo | Documentación inicial de la visión general del motor Dinámico. |
