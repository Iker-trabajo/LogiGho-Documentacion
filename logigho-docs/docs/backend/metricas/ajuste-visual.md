# Métricas — Ajuste visual del porcentaje (spec 005)

Gerencia pidió poder **elevar de forma uniforme los porcentajes de entrega que ven los usuarios**, sin cambiar el orden de las transportadoras, porque los porcentajes reales de producción salen bajos por la alta devolución. El administrador define un porcentaje mínimo en el panel y el sitio lo aplica al mostrar.

!!! warning "Decisión de negocio"
    Es una orden de gerencia (explicada por Iker, 2026-10-29). Lo que ve el usuario **no es el porcentaje real** aunque la pantalla lo rotule como "% de entrega efectiva". Se recomendó una nota visible en la vista; hoy no está. El ajuste se puede apagar en cualquier momento sin tocar datos.

## Principio: el dato real nunca se toca

| Capa | Qué guarda o calcula |
|---|---|
| Base de datos (`MetricasEntregaCiudadMes`, corridas) | **Porcentaje real** y conteos reales |
| Backend (lambda) | **Real**. Solo **guarda y entrega** los parámetros del ajuste |
| Front (vista de usuarios) | Aplica el ajuste **al mostrar** |
| Front (panel admin) | Muestra **real → mostrado** para revisar |

Es decisión de Iker (2026-10-29): "el backend no se debe modificar, los cálculos y la DB guardan el real; el porcentaje solo afecta al frontend".

## La regla

Deducida de la pizarra de gerencia (65 → 99, 50 → 90, 20 → 60, tope 99) y confirmada con lo explicado por Iker:

1. El administrador define `porcentajeMinimo` (1 a 99).
2. `desplazamiento = max(0, porcentajeMinimo − menor % real del grupo)`.
3. `mostrado = min(99, real + desplazamiento)`, con **un decimal**.

Ejemplo con mínimo 60: reales **65, 50 y 20** → desplazamiento 40 → **99** (65 + 40 = 105, tope), **90** y **60**. Coincide con la pizarra.

- El **orden nunca cambia** y las diferencias entre transportadoras se conservan hasta chocar con el tope de 99.
- Un porcentaje `null` (sin guías finales) se queda `null`.
- Si el mínimo definido no supera al menor real, no hay desplazamiento (solo se aplica el tope).
- Solo las celdas con **muestra suficiente** (`Finales ≥ MinimoGuiasPorDefecto`) definen el menor real; las demás reciben el mismo desplazamiento.

### Grupo sobre el que se calcula (`modo`)

| Modo | Qué es |
|---|---|
| `POR_CIUDAD` (por defecto) | Un desplazamiento por ciudad: las transportadoras de una misma ciudad son el "1, 2, 3" de la pizarra |
| `GLOBAL` | Un solo desplazamiento para todas las celdas del resultado |

!!! note "Hipótesis a confirmar con gerencia"
    Se asumió que el "1, 2, 3" de la pizarra son las transportadoras de una ciudad. La pestaña **Ajuste de porcentajes** del panel permite ver ambos modos con datos reales antes de activar.

## Parámetros (`AjusteVisual` en la configuración)

| Campo | Valores | Por defecto |
|---|---|---|
| `activo` | `true` / `false` | `false` |
| `porcentajeMinimo` | entero 1 a 99 | `60` |
| `modo` | `POR_CIUDAD` / `GLOBAL` | `POR_CIUDAD` |

- Se guardan en el documento `_id = "entrega-ciudad"` de `MetricasEntregaConfiguracion` (campo `AjusteVisual`).
- `PUT /configuracion` los valida: mínimo fuera de 1-99 o modo inválido → **400**.
- `GET /metricas/entrega-ciudad` los devuelve en `ajusteVisual` para que cualquier usuario autenticado pueda aplicarlos (la ruta de configuración es solo de admin).

## Dónde se aplica en el front

- **Vista de usuarios (Entregas):** `MetricasEntregaService.obtenerResumenParaUsuario()` descarga el resumen real, lee `ajusteVisual` y llama a `aplicarAjuste` (`views/tienda/entregas/utils/ajuste.utils.ts`) **antes** de pintar. Así el semáforo, la mejor opción, los rankings y la exportación a Excel usan los números ajustados. Cada celda conserva el real en `porcentajeReal`.
- **Panel de administración:** la pestaña *Ajuste de porcentajes* usa el resumen real y calcula la vista previa con los valores **sin guardar** (aunque el ajuste esté apagado), con búsqueda y promedios de real y mostrado.

## Cómo activarlo en producción

1. Entrar al panel (CEO o Desarrollador) → **Ajuste de porcentajes**.
2. Elegir el modo y el mínimo y revisar la vista previa con gerencia.
3. Marcar **Activo** y pulsar **Guardar ajuste**. Los usuarios lo ven al recargar Entregas.
4. Para volver al dato real: **Apagado** y guardar.

## Pruebas

- Front: 14 tests de `ajustarValores` / `aplicarAjuste` (la pizarra 65-50-20, tope, nulos, muestra, ambos modos, no muta la entrada).
- Backend: validación de la configuración (mínimo y modo) y que el resumen entrega `ajusteVisual` **sin** modificar los porcentajes.
