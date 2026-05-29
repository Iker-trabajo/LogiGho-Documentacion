
## 4. Componente: ProductosResumenTablaComponent

**Selector:** `app-productos-resumen-tabla`
**Ubicación:** `src/app/views/logistica/resumen-inventario/components/productos-resumen-tabla/productos-resumen-tabla.component.ts`
**Acceso:** Sección central de la vista principal de Resumen Inventario

---

### ¿Qué hace? (para el usuario)

Muestra el desglose completo de todos los productos del inventario en una tabla paginada. Por cada producto el operario puede ver:

- ID, nombre, variación, color, tienda
- Entradas, anuladas, salidas, stock actual, consumo, entregas, tránsito
- Costo del inventario físico, inventario rodante y total en pesos
- Estado del stock con color (verde, amarillo, rojo) relativo al mínimo configurado
- El stock mínimo configurado por producto con opción de editarlo inline

El usuario puede:
- Filtrar por texto (nombre, ID, tienda)
- Filtrar por tienda con un dropdown
- Paginar (10, 25, 50 o 100 productos por página)
- Exportar la vista actual a Excel

---

### Decoradores y configuración técnica

```typescript
@Component({
  selector: 'app-productos-resumen-tabla',
  standalone: true,
  imports: [CommonModule, FormsModule],
})
export class ProductosResumenTablaComponent implements OnChanges
```

Implementa `OnChanges` para resetear la paginación a la página 1 cada vez que cambian los productos recibidos por `@Input()`.

---

### Propiedades clave

**Inputs/Outputs:**

| Propiedad | Tipo | Descripción |
|---|---|---|
| `@Input() productos` | `ResumenItem[]` | Lista completa de productos (ya filtrada por el padre) |
| `@Input() isLoading` | `boolean` | Muestra spinner mientras cargan los datos |
| `@Input() estadoMap` | `Map<string, string>` | IdProducto → "Activo" o "Inactivo" |
| `@Input() stockMinimoMap` | `Map<string, {_id, StockMinimo}>` | Stock mínimo configurado por producto |
| `@Output() guardarStockMinimo` | `EventEmitter<{IdProducto, StockMinimo}>` | Avisa al padre que el usuario quiere guardar un mínimo |

**Estado interno:**

| Propiedad | Tipo | Descripción |
|---|---|---|
| `busqueda` | `string` | Texto del buscador |
| `filtroTienda` | `string` | Tienda seleccionada en el dropdown |
| `pageIndex` | `number` | Página actual (base 0) |
| `pageSize` | `number` | Productos por página (default 25) |
| `editandoMap` | `Map<string, number>` | IdProducto → valor nuevo mientras el usuario escribe el mínimo |
| `inputGuardado` | `Set<string>` | IDs de productos guardados exitosamente (activa badge verde por 1.5s) |

**Getters:**

| Getter | Retorna | Descripción |
|---|---|---|
| `tiendas` | `string[]` | Lista única de tiendas, ordenada alfabéticamente |
| `datosFiltrados` | `ResumenItem[]` | Productos aplicando filtro tienda + búsqueda + estado (excluye Inactivos) |
| `datosPaginados` | `ResumenItem[]` | Slice de `datosFiltrados` para la página actual |
| `totalPaginas` | `number` | `ceil(datosFiltrados.length / pageSize)` |
| `desdeHasta` | `{desde, hasta, total}` | Texto "Mostrando 26-50 de 300" |
| `totales` | `objeto` | Suma de todos los campos numéricos de `datosFiltrados` |

**Métodos de cálculo financiero:**

| Método | Fórmula |
|---|---|
| `getCostoInventario(p)` | `p.Actual × PrecioProveedor` |
| `getInventarioRodante(p)` | `p.EnTransito × PrecioProveedor` (nunca negativo, mínimo 0) |
| `getTotalInventario(p)` | `getCostoInventario + getInventarioRodante` |

**Métodos de edición de stock mínimo:**

| Método | Descripción |
|---|---|
| `onInputStockChange(p, valor)` | Valida y guarda el valor en `editandoMap` mientras el usuario escribe |
| `confirmarStockMinimo(p)` | Emite `guardarStockMinimo` con el valor de `editandoMap` y activa badge de confirmación |
| `cancelarEdicion(p, input)` | Limpia `editandoMap` y restaura el input al valor guardado en DB |
| `isEditando(p)` | `true` si el producto tiene cambios pendientes en `editandoMap` |
| `isGuardado(p)` | `true` si el producto acaba de ser guardado (dura 1.5 segundos) |

---

### Servicios y endpoints

No llama al backend directamente. Las mutaciones se delegan al padre mediante `@Output() guardarStockMinimo`.

Usa la librería `xlsx` para exportar a Excel.

---

### Flujo principal

```
Padre envía productos[] + mapas por @Input()
  └─► ngOnChanges() → pageIndex = 0

Usuario escribe en el buscador
  └─► onBusquedaChange() → pageIndex = 0
      datosFiltrados recalcula → datosPaginados muestra página 1

Usuario edita stock mínimo de un producto
  └─► onInputStockChange() → editandoMap.set(id, valor)
      confirmarStockMinimo() → guardarStockMinimo.emit()
        └─► Padre ejecuta PUT o POST al backend
```

---

### Historial de cambios

| Fecha | Cambio |
|---|---|
| 2026-05-26 | Creación con paginación, filtros y stock mínimo editable inline |

---

### Observaciones

- La tabla siempre **excluye productos Inactivos** (si `estadoMap` tiene datos). El filtro de estado del header solo controla el padre; la tabla tiene su propia capa de exclusión.
- `getColorStock()` devuelve `'neutro'` cuando el stock mínimo es 0, evitando alarmas falsas en productos sin configuración.
- El badge de confirmación usa `setTimeout` de 1500ms para darse de baja automáticamente del Set `inputGuardado`.
