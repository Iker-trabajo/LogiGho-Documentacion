## Autor: Iker Acevedo

Fecha creacion: 2026-08-26

Estado: produccion

## Componente: filtro-multiple

**Selector:** `app-filtro-multiple`
**Ubicación:** `components/filtro-multiple/`

---

## ¿Qué hace?

Dropdown de selección múltiple con buscador, genérico a propósito (recibe una lista de textos por `@Input`) — hoy lo usa el filtro de transportadoras dentro de [`detalle-lote`](detalle-lote.md), pero sirve igual para tienda, motivo o cualquier lista de opciones futura sin duplicar el dropdown. Sigue el mismo patrón visual (`.filtro-btn`/`.filtro-dropdown`) que los filtros de `resumen-inventario` y `cambio-entrega-inter`, para que el sitio se sienta uno solo.

```
@Input() etiqueta, placeholderBusqueda, opciones: string[]
@Input() set seleccionadas(valores: string[])   controlado por el padre — asi "limpiar filtros" tambien lo vacia
@Output() cambio: EventEmitter<string[]>
```

---

## `ChangeDetectionStrategy.OnPush`

El contenido solo se repinta cuando cambian sus `@Input` o el usuario interactúa. Sin esto, se repintaría en cada ciclo de detección de cambios de toda la página — con la tabla de `detalle-lote` abierta (potencialmente miles de filas), eso es constante.

---

## Cierre al hacer clic fuera

```typescript
@HostListener('document:click', ['$event'])
alClicFuera(evento) { if (abierto && !host.nativeElement.contains(evento.target)) abierto = false; }
```

Patrón estándar de dropdown — sin esto, el menú quedaría abierto hasta que el usuario lo cierre explícitamente.

---

## Por qué emite una copia del `Set`, no la referencia

```typescript
this.cambio.emit(Array.from(this.seleccionLocal));
```

Si se emitiera directamente el `Set` interno, el padre podría mutarlo por accidente y descuadrar el estado del dropdown sin que ninguno de los dos lados se entere.
