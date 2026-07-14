# Componente: ModernTableComponent

---

## Autor: Adalberto González
Fecha creación: 2026-07-04
Estado: desarrollo
Tipo: componente base reutilizable

---

## ¿Qué hace? (para el usuario)

Tabla genérica que cualquier módulo puede usar pasándole filas y columnas. Incluye de serie:

- **Búsqueda full-text** en tiempo real sobre todos los campos de cada fila.
- **Paginación** con selector de tamaño de página (10 / 25 / 50).
- **Selección por checkbox** (opcional) con banner de conteo y select-all en cabecera.
- Footer con color de marca, contador de resultados y paginador.

---

## Selector

```
app-modern-table
```

**Ubicación:** `src/app/views/base/modern-table/modern-table.component.ts`

---

## Decoradores

| Decorador | Nombre | Tipo | Descripción |
|---|---|---|---|
| `@Input` | `rows` | `any[]` | Datos a mostrar. Al cambiar reinicia búsqueda, página y selección |
| `@Input` | `columns` | `ModernTableColumn[]` | Definición de columnas (key, title, transform opcional) |
| `@Input` | `searchPlaceholder` | `string` | Placeholder del buscador. Default: `'Buscar...'` |
| `@Input` | `pageSize` | `number` | Filas por página iniciales. Default: `10` |
| `@Input` | `selectable` | `boolean` | Activa columna de checkboxes. Usa `_id` como clave |
| `@Input` | `selectionBannerText` | `string` | Texto del banner de selección. Vacío = sin banner |
| `@Output` | `seleccionCambio` | `EventEmitter<any[]>` | Emite las filas seleccionadas al cambiar la selección |

---

## Interfaz ModernTableColumn

```typescript
export interface ModernTableColumn {
  key: string;                        // Campo del objeto a mostrar
  title: string;                      // Encabezado visible en la tabla
  transform?: (value: any) => string; // Formatea el valor antes de mostrarlo
}
```

**Ejemplo con transform:**
```typescript
{ key: 'TotalRecaudo', title: 'Recaudo',
  transform: (v: string) => `$${Number(v || 0).toLocaleString('es-CO')}` }
```

---

## Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `filteredRows` | `any[]` | Resultado de aplicar `searchTerm` sobre `rows` |
| `seleccionadas` | `Set<string>` | IDs (`_id`) de las filas seleccionadas actualmente |
| `currentPage` | `number` | Página activa del paginador |
| `pageSizeOptions` | `number[]` | Opciones del selector: `[10, 25, 50]` |

**Getters computados:**

| Getter | Descripción |
|---|---|
| `registrosPaginados` | Slice de `filteredRows` para la página actual |
| `totalPaginas` | `Math.ceil(filteredRows.length / pageSize)` |
| `paginasVisibles` | Números de página con elipsis (`-1`) para el paginador compacto |
| `todosSeleccionados` | `true` si todas las filas filtradas están en `seleccionadas` |
| `algunoSeleccionado` | `true` si hay selección parcial (estado indeterminado del checkbox) |

---

## Flujo principal

```
@Input rows cambia
  └─► ngOnChanges → resetea búsqueda, página y selección → applyFilter()

Usuario escribe en el buscador
  └─► applyFilter() → filtra por JSON.stringify(row).includes(term) → currentPage = 1

Usuario cambia de página / tamaño
  └─► cambiarPagina() / cambiarSizePagina() → registrosPaginados recalcula

Usuario activa checkbox de fila
  └─► toggleSeleccion() → actualiza Set<_id> → emitirSeleccion()
        └─► seleccionCambio.emit(rows filtrados por Set)

Usuario activa checkbox de cabecera
  └─► toggleTodos() → clear + add all filteredRows → emitirSeleccion()
```

---

## Uso típico

```html
<app-modern-table
  [rows]="misFila"
  [columns]="columnas"
  [pageSize]="10"
  [selectable]="true"
  selectionBannerText="registros seleccionados"
  searchPlaceholder="Buscar por guía, tienda..."
  (seleccionCambio)="onSeleccion($event)"
></app-modern-table>
```

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-03 | Adalberto González | Creación con búsqueda, paginación y selección por checkbox |

---

## Observaciones

- La búsqueda usa `JSON.stringify(row)` para filtrar sobre todos los campos sin necesidad de declarar cuáles son buscables. Funciona bien para tablas de hasta ~500 filas; para volúmenes mayores considerar filtrado en backend.
- `seleccionadas` es un `Set<string>` de `_id`. Si las filas no tienen `_id` y `selectable = true`, la selección no funcionará. Todas las colecciones de MongoDB devuelven `_id` por defecto.
- `emitirSeleccion` filtra desde `rows` (fuente completa), no desde `filteredRows`, por lo que las filas seleccionadas en páginas anteriores se conservan aunque el buscador las oculte.
- `paginasVisibles` inserta `-1` como marcador de elipsis; el template lo renderiza como `...` en vez de un botón.
