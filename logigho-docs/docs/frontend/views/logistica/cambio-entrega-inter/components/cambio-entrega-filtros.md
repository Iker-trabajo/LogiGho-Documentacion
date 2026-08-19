---

## Autor:
Fecha creacion: 2026-08-20
Estado: produccion
Tipo: componente

# Componente: CambioEntregaFiltrosComponent

**Selector:** `app-cambio-entrega-filtros`
**Ubicación:** `src/app/views/logistica/cambio-entrega-inter/components/cambio-entrega-filtros/`

---

## ¿Qué hace?

Panel de filtros de la tabla: tienda/ecosistema, búsqueda libre, estado de gestión, y **2 calendarios independientes** (fecha de detección y fecha de creación del pedido). Reutiliza `filtro-tienda-ecosistema` y `rango-fecha-calendario` (dos instancias) en vez de reimplementar dropdowns propios.

---

## Decoradores

| Decorador | Nombre | Tipo | Descripción |
| --------- | ------ | ---- | ----------- |
| `@Input` | `filtros` | `CambioEntregaFiltros` | Estado controlado por el padre |
| `@Input` | `fechasDisponibles` | `string[]` | Días con al menos una detección — habilita esos días en el calendario de "Fecha cambio de estado" |
| `@Input` | `fechasCreacionDisponibles` | `string[]` | Igual, para "Fecha de creación" |
| `@Input` | `tiendaMap` | `Map<string, string[]>` | Ecosistema → Tiendas |
| `@Output` | `filtrosChange` | `EventEmitter<Partial<CambioEntregaFiltros>>` | Emite solo lo que cambió |

---

## Flujo principal

```
onRangoFecha(seleccion)          -> filtrosChange.emit({fechaDesde, fechaHasta})
onRangoFechaCreacion(seleccion)  -> filtrosChange.emit({fechaCreacionDesde, fechaCreacionHasta})
onTiendasChange(tiendas)         -> filtrosChange.emit({tiendasSeleccionadas})
onBusquedaChange(valor)          -> filtrosChange.emit({busqueda})
onEstadoChange(valor)            -> filtrosChange.emit({estadoGestion})
limpiarTodo()                    -> filtrosChange.emit(estado inicial completo)
```

`hayFiltrosActivos` (getter) controla si se muestra el botón "Limpiar filtros" — considera los 2 rangos de fecha por separado.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-31 | Iker Acevedo | Filtros inicial: tienda/ecosistema, búsqueda, estado, 1 rango de fecha. |
| 2026-08-18 | Iker Acevedo | Segundo `rango-fecha-calendario` (fecha de creación del pedido), cada uno con su propia `etiquetaBase` para que el usuario sepa a cuál fecha filtra sin necesidad de un label externo. |
| 2026-08-20 | Iker Acevedo | Fix de un bug real: el `<app-rango-fecha-calendario>` se estiraba verticalmente de forma impredecible por un `flex: 1 1 200px` que asumía ser hijo directo de una fila — al envolverlo en un `<div>` para el label, ese mismo flex ahora actuaba sobre el eje vertical. El fix de layout va en el wrapper, no en el calendario. `box-sizing: border-box` en el `<select>` de Estado de gestión (se desbordaba del contenedor en mobile). `font-size: 16px` en mobile (fix del zoom automático de iOS Safari). |

---

## Observaciones

- Los 2 calendarios comparten el mismo componente reutilizable (`rango-fecha-calendario`) con `etiquetaBase` distinta — no hay 2 implementaciones de calendario en el módulo.
- Un bug de referencia inestable en `seleccionadasCalendario`/`seleccionadasCalendarioCreacion` (getters que devuelven un array nuevo en cada chequeo de cambios) causaba que el calendario pareciera "no responder a ningún clic" — se corrigió en el componente `rango-fecha-calendario` comparando por valor contra la última selección aplicada, no por referencia. Ver [Tour Guiado](../../../../components/guided-tour.md) para el mismo patrón de bug en otro componente del módulo.
