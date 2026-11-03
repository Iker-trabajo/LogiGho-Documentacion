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
2. `desplazamiento = porcentajeMinimo − menor % real del grupo` (puede ser negativo: la fórmula **siempre se aplica**).
3. `mostrado = min(99, real + desplazamiento)`, con **un decimal** (nunca por debajo de 0).

Ejemplo con mínimo 60: reales **65, 50 y 20** → desplazamiento 40 → **99** (65 + 40 = 105, tope), **90** y **60**. Coincide con la pizarra.

### La misma fórmula, en escalones (como la explica gerencia)

1. El **peor** del grupo (menor porcentaje real con muestra suficiente) se muestra con el **mínimo**.
2. Cada transportadora = **mínimo + lo que le saca de ventaja al peor** (su real − el real del peor).
3. Si pasa de **99 %**, se muestra 99 % (tope).

Con mínimo 60 y reales 65, 50 y 20: A = 60 + (65 − 20) = 105 → **99 %** (tope), B = 60 + (50 − 20) = **90 %**, C = 60 + 0 = **60 %**. Es matemáticamente igual a `real + (mínimo − peor)` (un test lo comprueba en varios casos); `escalonar` en `ajuste.utils.ts` es la versión paso a paso que usa la calculadora.

**Calculadora del panel:** dentro de "Cómo funciona la fórmula" hay una tabla con las transportadoras A, B y C en el mismo formato de la vista previa (`real % → mostrado`, reales editables, por defecto 65, 50 y 20) y debajo el cálculo paso a paso. Se recalcula con el mínimo que escribe el administrador. Es solo de práctica: no guarda nada.

- El **orden nunca cambia** y las diferencias entre transportadoras se conservan hasta chocar con el tope de 99.
- Un porcentaje `null` (sin guías finales) se queda `null`.
- **Si el mínimo es menor que el peor real, todo baja** los mismos puntos (con mínimo 10, los reales 65, 50 y 20 se muestran 55, 40 y 10). El peor siempre queda en el mínimo.
- En una ciudad con una sola transportadora con muestra, esa transportadora muestra exactamente el mínimo (en producción, 843 de 1 140 celdas con muestra están en ciudades así). Para esas ciudades conviene un porcentaje propio.
- Solo las celdas con **muestra suficiente** (`Finales ≥ MinimoGuiasPorDefecto`) definen el menor real; las demás reciben el mismo desplazamiento.

### Grupo sobre el que se calcula (`modo`)

| Modo | Qué es |
|---|---|
| `POR_CIUDAD` (por defecto) | Un desplazamiento por ciudad: las transportadoras de una misma ciudad son el "1, 2, 3" de la pizarra |
| `GLOBAL` | Un solo desplazamiento para todas las celdas del resultado |

**¿Cuál elegir?** Por ciudad mantiene a cada ciudad con su peor transportadora en el mínimo. Global usa el peor porcentaje de todo el país: si ese valor es muy bajo, el desplazamiento es grande y casi todo choca con el tope de 99 %, con lo que se pierden las diferencias. Se recomienda `POR_CIUDAD`; la vista previa del panel permite compararlos.

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

## Ciudades: general para todas y personalizado por ciudad

Por defecto **todas las ciudades usan el porcentaje mínimo general**. En el panel, cada ciudad tiene una casilla **General**:

| Casilla | Efecto |
|---|---|
| **Marcada** (por defecto) | La ciudad usa el mínimo general |
| **Desmarcada** | Se habilita **Mínimo propio** (entero de 1 a 99, arranca igual al general) y la ciudad se calcula de una con ese valor; aparece un ✔ verde que confirma que tiene un valor personalizado |

- Solo se guardan las ciudades **personalizadas**, en `AjusteVisual.Excepciones` (`{ Mun8, Excluida, PorcentajeMinimo }`; el panel solo escribe `PorcentajeMinimo`). `Excluida: true` (mostrar el real) sigue siendo válido en el backend y en el cálculo, pero el panel ya no lo ofrece. Una lista vacía significa "todas con el general".
- El `PUT /configuracion` rechaza con 400 un `mun8` inexistente, una ciudad repetida o un porcentaje propio fuera de 1-99.
- En modo `GLOBAL`, las ciudades con porcentaje propio **no** cuentan para calcular el desplazamiento general.
- Atajos (siguen al buscador): *Usar el general en las que se ven*, *Personalizar las que se ven* y *Restablecer ciudades*, con un contador de ciudades con porcentaje propio.
- **Ajuste apagado:** todo se desactiva con el mensaje "el porcentaje no se puede configurar". El checkbox **"Solo para calcular"** habilita los controles para probar valores en la vista previa sin aplicar nada a los usuarios.
- Las cajas de porcentaje solo aceptan enteros de 1 a 99 (se acota lo que se sale y se bloquean signos, punto y coma).

!!! warning "Orden de despliegue"
    La lambda anterior a esta versión **ignora** las excepciones (no las guarda). Si el front se despliega antes, el panel avisa con "la lambda desplegada todavía no guarda las excepciones por ciudad". Desplegar primero la lambda.

## Dónde se aplica en el front

- **Vista de usuarios (Entregas):** `MetricasEntregaService.obtenerResumenParaUsuario()` descarga el resumen real, lee `ajusteVisual` y llama a `aplicarAjuste` (`views/tienda/entregas/utils/ajuste.utils.ts`) **antes** de pintar. Así el semáforo, la mejor opción, los rankings y la exportación a Excel usan los números ajustados. Cada celda conserva el real en `porcentajeReal`.
- **Panel de administración:** la pestaña *Ajuste de porcentajes* usa el resumen real y calcula la vista previa con los valores **sin guardar** (aunque el ajuste esté apagado), con búsqueda y promedios de real y mostrado.

## Cómo activarlo en producción

1. Entrar al panel (CEO o Desarrollador) → **Ajuste de porcentajes**.
2. Elegir el modo y el mínimo y revisar la vista previa con gerencia.
3. Marcar **Activo** y pulsar **Guardar ajuste**. Es **inmediato**: no espera ninguna corrida. Los usuarios lo ven al recargar Entregas (el resumen trae `ajusteVisual` en cada lectura).
4. Para volver al dato real: **Apagado** y guardar.

## Pruebas

- Front: tests de `ajustarValores` / `aplicarAjuste` (la pizarra 65-50-20, tope, nulos, muestra, ambos modos, no muta la entrada) y de las excepciones por ciudad (excluir, porcentaje propio, GLOBAL, edición de la lista).
- Backend: validación de la configuración (mínimo, modo y excepciones por ciudad) y que el resumen entrega `ajusteVisual` **sin** modificar los porcentajes.
