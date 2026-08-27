## Autor:
Fecha creacion: 2026-08-26
Estado: aceptada

# ADR-004 — Fechas propias del módulo en UTC real, con una excepción deliberada

**Autor:** Iker Acevedo
**Fecha:** 2026-08-26
**Estado:** Aceptada

---

## Contexto

`GuiaProcesada.Fecha` se escribía con `DateTime.UtcNow.AddHours(-5)` — un `DateTime` con `Kind=Utc` que en realidad contenía números de hora Colombia. Mientras tanto, `JobDevolucion.FechaInicio` sí usaba `DateTime.UtcNow` sin ajustar, UTC real.

El resultado: el mismo instante se veía con **5 horas de diferencia** según qué colección lo leyeras. Comparando datos reales de un mismo job:

```
JobsDevolucion.FechaInicio        2026-08-26T00:24:32Z   (UTC real)
InventarioDevolucion.Fecha        2026-08-25T19:24:34Z   (Colombia disfrazada de UTC)
```

Cualquier consumidor que tratara ese campo como UTC real (el driver de Mongo, `System.Text.Json`, el pipe `date` de Angular) le restaría 5 horas **otra vez**, sumando el error.

---

## Opciones consideradas

### Opción A — Todo en UTC real, conversión a hora Colombia solo en el front

`GuiaProcesada.Fecha = DateTime.UtcNow` (sin restar horas). La conversión a hora Colombia se hace una sola vez, en el front, al momento de mostrarla (`aHoraColombia`, `fechaColombiaDesdeJob`/`fechaColombiaDesdeInventario`).

**Pros:** un `DateTime` con `Kind=Utc` que de verdad contiene UTC es la única forma de que todo el ecosistema (Mongo, serialización, otros consumidores futuros) lo interprete de forma consistente. La conversión a hora local es responsabilidad de la capa de presentación, no del dato persistido — es la práctica estándar.
**Contras:** las filas de `InventarioDevolucion` escritas **antes** de este cambio quedan con hora Colombia disfrazada de UTC — se van a mostrar 5 horas antes de lo real hasta que se corrijan (no se hizo backfill en esta iteración, son las guías de las pruebas iniciales).

### Opción B — Todo en hora Colombia, consistente

Cambiar también `JobDevolucion.FechaInicio` a `UtcNow.AddHours(-5)`, para que las dos colecciones coincidan en "hora Colombia disfrazada de UTC".

**Pros:** no requiere backfill de nada — todo lo existente ya está en ese esquema.
**Contras:** perpetúa el problema de fondo: un `DateTime` marcado `Kind=Utc` que no es UTC real sigue siendo una mentira sobre el propio dato, y cualquier integración futura con un sistema externo que sí respete `Kind=Utc` (o cualquier reporte/BI que lea Mongo directo) heredaría el error silenciosamente.

---

## Decisión

**Se eligió:** Opción A, con una excepción deliberada.

**La excepción:** `PedidosInter."Fecha Dev Completada"` (escrita por `DevolucionRepository.MarcarDevolucionCompletadaAsync`) se mantiene en **hora Colombia**, no UTC. Esa colección es compartida con toda la plataforma LogiGho, y el módulo legacy ya escribe ese campo en hora local — pasarlo a UTC dejaría las filas de este módulo desalineadas 5 horas contra todo lo demás que lee ese mismo campo en `PedidosInter`. Consistencia con el resto de la plataforma le gana a pureza técnica dentro de una colección compartida.

**Razón general:** la conversión de zona horaria debe ocurrir en un solo lugar (el borde de presentación), no en cada escritura del backend. Con UTC real, cualquier consumidor nuevo que respete la convención estándar (`Kind=Utc` significa UTC) funciona correctamente sin sorpresas.

---

## Consecuencias

**Positivas:** `JobsDevolucion.FechaInicio` e `InventarioDevolucion.Fecha` ahora son directamente comparables — el mismo instante se ve igual en ambas colecciones. El front tiene un único punto de conversión (`aHoraColombia` en `ingreso-devoluciones.rules.ts`), y no depende del pipe `date` de Angular (que convertiría según la zona horaria del **navegador**, mostrando horas distintas según desde dónde se abra el sitio — la operación es colombiana, la hora mostrada debe ser siempre la de Colombia).

**Negativas:** las filas de `InventarioDevolucion` escritas antes de este cambio se muestran 5 horas antes de lo real. Documentado en el código (`rules.ts`) para que quien las vea entienda por qué, sin necesidad de un backfill inmediato — son datos de las pruebas iniciales del módulo, no de producción real en volumen.

---

## Impacto en el código

| Componente | Cambio |
| ---------- | ------ |
| `GuiaProcesada.cs` | `Fecha = DateTime.UtcNow` (antes `.AddHours(-5)`) |
| `DevolucionRepository.MarcarDevolucionCompletadaAsync` | Comentario explícito documentando la excepción deliberada de `"Fecha Dev Completada"` |
| `ingreso-devoluciones.rules.ts` (front) | `fechaColombiaDesdeJob`, `fechaColombiaDesdeInventario`, `formatFechaLarga` — conversión y formato en un solo lugar, con accesores `getUTC*` a propósito (la fecha ya viene en hora Colombia al llegar a esta función) |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-08-26 | Iker Acevedo | Decisión inicial, cambio de `GuiaProcesada.Fecha` a UTC real, funciones de conversión en el front. |

---

## Referencias

- [`ApiLambdaDevolucionesMasivo.md`](../ApiLambdaDevolucionesMasivo.md) — variables de entorno y flujo general.
- [`ingreso-devoluciones-rules.md`](../../../../../frontend/views/logistica/ingreso-devoluciones/helpers/ingreso-devoluciones-rules.md) — funciones de conversión de fecha en el front.
