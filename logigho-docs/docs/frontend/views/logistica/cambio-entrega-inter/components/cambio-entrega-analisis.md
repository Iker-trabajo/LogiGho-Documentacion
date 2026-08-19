---

## Autor:
Fecha creacion: 2026-08-20
Estado: produccion
Tipo: componente

# Componente: CambioEntregaAnalisisComponent

**Selector:** `app-cambio-entrega-analisis`
**Ubicación:** `src/app/views/logistica/cambio-entrega-inter/components/cambio-entrega-analisis/`

---

## ¿Qué hace?

Modal de análisis con 3 gráficas (Chart.js vía `@coreui/angular-chartjs`), todas calculadas sobre el mismo set de datos ya filtrado por rango de fecha y, opcionalmente, por ecosistema. No pide nada nuevo al backend — recibe `items` (la colección completa que ya cargó el componente raíz) y calcula todo en memoria.

---

## Decoradores

| Decorador | Nombre | Tipo | Descripción |
| --------- | ------ | ---- | ----------- |
| `@Input` | `isOpen` | `boolean` | Abre/cierra el modal |
| `@Input` | `items` | `CambioEntregaInter[]` | Dataset completo (sin paginar) que ya tiene el padre |
| `@Input` | `tiendaMap` | `Map<string, string[]>` | Ecosistema → Tiendas — para el selector de ecosistema y el cruce `item.Tienda -> ecosistema` |
| `@Input` | `ecosistemaInicial` | `string \| null` | Ecosistema a preseleccionar **al abrir** (ver más abajo) |
| `@Output` | `cerrar` | `EventEmitter<void>` | Cierre del modal |

---

## Las 3 gráficas

| Pestaña | Tipo | Qué muestra |
| ------- | ---- | ----------- |
| Distribución | Doughnut | % de Pendiente / Entrega / Devolución sobre el total en rango |
| Tendencia | Barras | Guías detectadas por día/semana/mes — la granularidad se adapta al rango (7-30 días = por día, 90 días = por semana, todo = por mes) |
| Top tiendas | Barras horizontales | Las 8 tiendas con más guías auditadas en el rango — la vista que apunta a "dónde reclamar primero" |

Todas se recalculan (`recalcular()`) cada vez que cambia el rango de fecha o el ecosistema — un solo método arma los 3 `ChartData` a la vez.

---

## Filtros dentro del modal

### Rango de fecha

7 / 30 / 90 días, o todo el histórico. Filtra sobre `FechaPrimeraDeteccion` (fecha de detección) — **no** sobre la fecha de creación del pedido, se aclara explícitamente en el texto del modal para no confundirlo con el filtro de la tabla principal.

### Ecosistema

Selector con el mismo look que `filtro-tienda-ecosistema` del panel principal (botón + panel flotante, no un `<select>` nativo) — cruza `item.Tienda` contra `tiendaMap.get(ecosistema)`. Al elegir uno, aparece un banner "Mostrando solo el ecosistema **X**" con botón "Limpiar" dentro del propio modal.

**Preselección automática:** si al abrir el modal todas las tiendas que ya están filtradas en la tabla principal pertenecen a un único ecosistema, ese ecosistema llega precargado (`ecosistemaInicial`, calculado por el componente raíz). Si el usuario tiene tiendas de varios ecosistemas mezcladas en el filtro de la tabla (o no filtró ninguna), no hay uno solo que inferir — el selector abre vacío y el usuario elige a mano. La preselección solo se aplica **al abrir** el modal, no se vuelve a pisar mientras el usuario lo tiene abierto y cambia el ecosistema manualmente.

---

## Flujo principal

```
ngOnChanges(changes)
  -> [isOpen: true] bloquea scroll de fondo
                     ecosistema = ecosistemaInicial
                     recalcular()
  -> [items cambia, modal ya abierto] recalcular()

recalcular()
  -> filtrarPorRango(items, rango)
  -> filtrarPorEcosistema(..., ecosistema)
  -> arma desenlaceResumen {pendiente, entrega, devolucion}
  -> construirDonut / construirTendencia / construirTiendas
```

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-08-18 | Iker Acevedo | Componente inicial: dona de distribución. |
| 2026-08-18 | Iker Acevedo | Se agregan Tendencia y Top tiendas — 3 pestañas en total. Selector de rango de fecha. |
| 2026-08-20 | Iker Acevedo | Selector de ecosistema (con preselección automática desde la tabla) + banner de filtro activo. Reemplazo del `<select>` nativo de ecosistema por el mismo look de `filtro-tienda-ecosistema`. Fix del número total desbordando el círculo de la dona en pantallas chicas/números grandes (`clamp()` de font-size + ancho constreñido al hueco real de la dona). |

---

## Observaciones

- **Bug de Chart.js resuelto:** los datos de cada gráfica (`donutData`, `tendenciaData`, `tiendaData`) son campos de clase reasignados **por completo** en `recalcular()`, nunca un `get` calculado en el template. Un `get` se reevalúa en cada chequeo de cambios de Angular y devuelve un objeto nuevo cada vez — CoreUI/Chart.js interpretaba esa referencia distinta como "los datos cambiaron" y reiniciaba la animación de entrada en bucle (la torta nunca terminaba de pintarse, quedaba titilando). Aplica el mismo principio a cualquier `[data]`/`[options]` que se le pase a `c-chart` en el resto de la plataforma.
- **`maintainAspectRatio: false` + contenedor de tamaño fijo real** en CSS (no solo `width:100%`) es obligatorio para `c-chart` — si el contenedor no tiene una altura real, Chart.js entra en un loop de resize.
- La regla de clasificación Entrega/Devolución/Pendiente es la misma que usa la tabla y el backend — no hay una segunda implementación de esa lógica en este componente, solo el filtrado por rango/ecosistema es propio de acá.
