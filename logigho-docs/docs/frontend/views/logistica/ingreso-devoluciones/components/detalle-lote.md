## Autor: Iker Acevedo

Fecha creacion: 2026-08-26

Estado: produccion

## Componente: detalle-lote

**Selector:** `app-detalle-lote`
**Ubicación:** `components/detalle-lote/`

---

## ¿Qué hace?

Detalle de un lote: guías aceptadas (de `InventarioDevolucion`) y rechazadas (del propio documento del lote), cada una en su pestaña, con buscador, filtro de transportadora, paginación y exportación a Excel. Recibe los datos por `@Input` — nunca lee del store directamente. Esto es lo que permite que [`historial-lotes`](historial-lotes.md) lo instancie una vez por cada fila desplegada, mostrando un lote distinto cada vez.

```
@Input() rechazadas: GuiaProcesada[]
@Input() aceptadas: InventarioDevolucionDoc[]
@Input() cargandoAceptadas: boolean
@Input() filtroGuiaInicial: string     — texto que baja desde el buscador global del historial
@Input() jobId: string                 — solo para nombrar el archivo exportado
@Output() reintentar: EventEmitter<string[]>
```

---

## Por qué todo se calcula en campos, nunca en getters del template — un bug real que esto arregló

**Este es el hallazgo más importante de este componente, documentado en la cabecera del código:**

Un getter llamado desde el template de Angular se reevalúa en **cada ciclo de detección de cambios** — incluido el que dispara **mover el mouse**. Como funciones como "rechazadas filtradas" o "resumen por familia" devuelven arreglos **nuevos** cada vez que se llaman, `*ngFor` sin `trackBy` veía esos arreglos como datos distintos en cada ciclo y **recreaba todo el DOM de la tabla**. El síntoma real: **el operario no podía seleccionar texto con el mouse para copiarlo** — la selección se perdía al instante porque el elemento seleccionado se destruía y se recreaba.

Además, con miles de filas, recalcular el resumen agrupado en cada movimiento del mouse era costoso de verdad, no solo un detalle cosmético.

**La solución:** un único método `recalcular()` que corre solo cuando de verdad cambia algo (`ngOnChanges`, cambio de filtro, cambio de página), guarda el resultado en campos normales de la clase, y el template lee esos campos — nunca invoca una función. Se combinó con `trackBy` (`trackPorGuia`, `trackPorMotivo`) para que Angular no recree filas que no cambiaron, y `user-select: text` explícito en el CSS.

---

## Flujo de filtrado

```
recalcular()  — el UNICO punto donde se recalculan los derivados
  transpSel = Set(transportadorasSeleccionadas)     O(1) por fila, no O(n) recorriendo el arreglo
  rechazadasFiltradas = rechazadas.filter(texto Y transportadora)
  aceptadasFiltradas  = aceptadas.filter(texto Y transportadora)
  unidadesRecuperadas = SUMA de TODAS las aceptadas filtradas (no solo la pagina visible)
  paginaRechazadas / paginaAceptadas = paginar(filtradas, pagina, tamanoPagina)
  resumen = resumenPorFamilia(rechazadas)            SIEMPRE sobre el total, no sobre el filtro
  transportadorasDisponibles = opciones del filtro, segun la pestaña activa
```

**`unidadesRecuperadas` suma sobre el filtro completo, no la página** — un total que cambiara al pasar de página no sería un total real.

**`exportarExcel()` exporta todo lo filtrado, no solo la página visible** — el operario espera el reporte completo de lo que ve filtrado, no solo las 25 filas en pantalla.

---

## Pestaña inicial: dónde está la guía que buscaste

```typescript
pestanaInicial()
  si filtroGuiaInicial coincide con alguna rechazada -> 'rechazadas'
  si coincide con alguna aceptada -> 'aceptadas'
  si no hay texto de busqueda -> 'rechazadas' si hay alguna, si no 'aceptadas'
```

Al abrir un lote desde el buscador global del historial, el detalle arranca ya filtrado por esa guía **y** parado en la pestaña donde esa guía realmente está — el operario nunca tiene que adivinar ni volver a escribir el filtro.

---

## Exportar a Excel — motivos traducidos

```typescript
Motivo: this.tituloMotivo(g.Motivo)          // "Pedido anulado", no "PedidoAnulado"
'Accion sugerida': infoMotivo(g.Motivo).accion
'Guia original': guiaOriginalDe(g) ?? ''
```

El reporte que se lleva el operario tiene lenguaje humano, nunca el nombre técnico del enum — la misma traducción que usa la UI.
