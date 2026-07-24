## Utilidad: resolver-corte

**Ubicación:** `src/app/views/tienda/pancake-paginas/utils/resolver-corte.ts`  
**Tipo:** funciones puras (sin clase, sin estado, sin inyección de Angular)

---

### ¿Qué hace?

Como se explica en [la vista principal](../pancake-paginas.md#cortes-del-dia-slots), los datos de gasto y resultados de un anuncio llegan varias veces al día en distintos cortes (slots). Este archivo tiene dos funciones puras que resuelven, cada una a su manera, qué hacer con las múltiples filas de un mismo anuncio.

---

### Funciones

#### `resolverFilaVigente(filas)`

Para un mismo anuncio, elige **una sola fila** — la más confiable — siguiendo este orden de prioridad:

1. Si existe el **cierre oficial** (slot `1`) del día más reciente, esa es la elegida. Se marca como `estadoCorte: 'cerrado'`.
2. Si no hay cierre pero sí una **verificación manual** (slot `99` o `5`), se usa esa. Se marca como `'verificado'`.
3. Si no hay ninguna de las dos, se usa el **corte parcial más avanzado del día** (entre `2`, `3` y `4`, el número más alto = la hora más tardía = el dato más completo hasta ese momento). Se marca como `'parcial'`.
4. Si no hay ninguna fila, devuelve `estadoCorte: 'sin_datos'`.

Se usa para calcular, por ejemplo, "¿cuánto ha gastado esta página hoy?" — sin esto, un mismo anuncio podría sumarse dos o tres veces si se contaran todas sus filas.

#### `resolverfilasParaComparacion(filas, slotIdsSeleccionados)`

A diferencia de la anterior, esta función **no elige una sola fila** — devuelve **una fila por cada slot pedido**, para poder compararlos lado a lado (por ejemplo, en un gráfico de barras que muestra "9 AM vs 2 PM vs 5 PM" del mismo día).

---

### Ejemplo sencillo

```
Filas de un anuncio el mismo día:
  slot 2 (9 AM):  gasto = 10.000
  slot 3 (2 PM):  gasto = 25.000
  slot 4 (5 PM):  gasto = 40.000

resolverFilaVigente(filas)
  → elige la del slot 4 (la más tardía = más completa)
  → { fila: {...gasto: 40.000}, estadoCorte: 'parcial', horaCorte: '5:00 PM' }

resolverfilasParaComparacion(filas, ['2', '3', '4'])
  → { '2': {...gasto: 10.000}, '3': {...gasto: 25.000}, '4': {...gasto: 40.000} }
  → sirve para graficar las 3 barras del día, una por corte
```

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-24 | Adalberto González | Documentación inicial de la utilidad, ya en su forma actual |

---

### Observaciones

- Estas funciones son **puras**: no tocan signals, no hacen HTTP, no dependen de Angular. Reciben datos y devuelven datos — por eso viven en `utils/` y no en el componente ni en el repositorio.
- Tienen su propio archivo de pruebas (`resolver-corte.spec.ts`), separado del resto del módulo, precisamente porque al ser funciones puras son fáciles de probar con distintos escenarios de filas.
- El componente `PancakePaginasComponent` usa `resolverFilaVigente()` para armar los KPIs y el resumen de campañas por página, y `resolverfilasParaComparacion()` para la tabla y el gráfico de "Comparativa de cortes".
