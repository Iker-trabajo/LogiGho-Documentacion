## Autor: Yadilton Torres

Fecha creacion: 2026-10-30

Estado: desarrollo

## Componente: filtro-desplegable

**Selector:** `app-filtro-desplegable`
**Ubicación:** `components/filtro-desplegable/`

---

## ¿Qué hace?

Desplegable propio para los filtros de Motivo, Estado, Tienda y Prioridad: muestra cuántas guías hay por opción, tiene buscador, ✓ en la elegida y una ✕ para quitar el filtro.

```
@Input() etiqueta, textoTodos = 'Todos'
@Input() opciones: OpcionFiltro[]      { valor, texto, contador?, tono?, detalle? }
@Input() valor                          controlado por el padre — así "Limpiar" también lo vacía
@Output() valorChange                   permite [(valor)]
```

---

## Contadores

El padre calcula el contador de cada opción **con los demás filtros puestos**, menos el propio, así el número es exactamente lo que sale al elegirla. Las opciones en cero se ven atenuadas (solo el texto; si están elegidas se ven normales) y van al final.

---

## Detalles de diseño

- **Espacio del ✓ siempre reservado:** si el ✓ apareciera solo en la opción elegida, empujaría el contador y las columnas se correrían.
- **Cierre:** al tocar fuera (`document:click`) o con Esc.
- **Celular (≤768px):** el menú se convierte en **hoja inferior** fija a la ventana, con agarradera, título, fondo oscuro detrás y controles de 44px. Así no lo cortan los contenedores con `overflow`. Es la misma técnica de `relacion-despacho.mobile.scss`.
