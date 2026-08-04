## Componente: ProductoSelectorModalComponent

**Selector:** `app-producto-selector-modal`  
**Ubicación:** `src/app/views/tienda/gestion-paginas/components/producto-selector-modal/producto-selector-modal.component.ts`  
**Acceso:** Se abre al hacer clic en el botón "Productos" de una fila de la tabla

---

### ¿Qué hace?

Modal para asociar uno o varios productos del catálogo (colección `Productos`) a una página. Tiene un buscador desplegable (mismo patrón que `AsignacionModalComponent` de `gestion-bodegas`): al escribir, aparece una lista con checkbox por cada producto que coincide. Debajo, en el cuerpo del modal, se muestra la lista de productos ya asociados a esa página, cada uno con un botón para quitarlo directamente. Al confirmar, guarda la lista completa seleccionada.

---

### Decoradores y configuración técnica

```typescript
@Component({
  selector: 'app-producto-selector-modal',
  standalone: true,
  imports: [CommonModule, FormsModule],
  templateUrl: './producto-selector-modal.component.html',
  styleUrl: './producto-selector-modal.component.scss'
})
export class ProductoSelectorModalComponent implements OnChanges
```

---

### Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `@Input() isOpen` | `boolean` | Controla visibilidad del modal |
| `@Input() productos` | `ProductoAsociado[]` | Catálogo completo de productos disponibles (cargado una sola vez por el padre) |
| `@Input() pagina` | `Pagina \| null` | Página a la que se le están asociando productos |
| `@Input() guardando` | `boolean` | `true` mientras se espera la respuesta del backend, deshabilita los botones |
| `@Output() guardar` | `EventEmitter<ProductoAsociado[]>` | Emite la lista final de productos seleccionados al confirmar |
| `@Output() cerrar` | `EventEmitter<void>` | Cierra el modal sin guardar |
| `buscarProducto` | `string` | Texto de búsqueda del selector desplegable |
| `mostrarMenuProducto` | `boolean` | Visibilidad del desplegable de búsqueda |
| `productosSeleccionados` | `Set<string>` | Ids (`idproducto`) actualmente marcados, fuente de verdad de la selección |
| `productosFiltrados` | `ProductoAsociado[]` | Resultado del catálogo filtrado por `buscarProducto`, para pintar el desplegable |
| `resumenProductosSeleccionados` | getter `ProductoAsociado[]` | Productos completos (no solo el id) que están seleccionados, usados para la lista del cuerpo del modal |

---

### Flujo principal

```
Padre abre el modal (modalProductosAbierto = true, paginaEnModalProductos = pagina)
  └─► ngOnChanges detecta isOpen: false→true
        └─► productosSeleccionados = ids de pagina.productosAsociados

Usuario escribe en el buscador de arriba
  └─► filtrarProductosDropdown() recalcula productosFiltrados
        └─► Aparece el desplegable con checkboxes

Usuario marca/desmarca un checkbox del desplegable
  └─► toggleProducto(producto) agrega o quita el id del Set
        └─► La lista de "productos asociados" (cuerpo del modal) se actualiza sola

Usuario hace clic en la "✕" de un producto ya asociado (en el cuerpo)
  └─► toggleProducto(producto) — mismo método, lo quita del Set

Usuario hace clic en "Guardar"
  └─► confirmar() → guardar.emit(resumenProductosSeleccionados)
        └─► El padre llama a repo.actualizarProductosAsociados(pagina._id, productos)
```

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-24 | Adalberto González | Creación del selector de productos asociados a una página, con el mismo estilo visual de `AsignacionModalComponent` (gestion-bodegas). Primera versión mostraba un chip por producto seleccionado y el SKU de cada uno; se simplificó a una lista tipo tarjeta (igual a `PaginaDetalleModalComponent`) mostrando solo el nombre, sin SKU |

---

### Observaciones

- El desplegable de búsqueda vive en una franja propia (`.selector-bar`) justo debajo del header del modal, **fuera** del cuerpo con scroll (`.modal-body`) — si estuviera dentro, el `overflow` del cuerpo recortaría el desplegable y no se vería completo al desplegarse.
- El checkbox de cada opción del desplegable usa `(change)` (no `(click)`) para marcar/desmarcar — usar `(click)` junto con `preventDefault()` causaba que el check visual no respondiera de inmediato al hacer clic directamente sobre la cajita.
- Guardar **reemplaza** la lista completa de productos asociados, no la combina con la anterior — por eso `resumenProductosSeleccionados` siempre se calcula sobre el catálogo completo (`productos`), no sobre lo que ya tenía la página.
- No muestra ni guarda `imagenUrl` ni `sku` de los productos — se decidió mostrar solo el nombre para mantener la lista simple y liviana.
