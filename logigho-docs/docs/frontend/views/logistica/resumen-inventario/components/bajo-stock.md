
## 3. Componente: BajoStockModalComponent

**Selector:** `app-bajo-stock-modal`
**Ubicación:** `src/app/views/logistica/resumen-inventario/components/bajo-stock-modal/bajo-stock-modal.component.ts`
**Acceso:** Se abre desde la tarjeta KPI "Bajo Stock"

---

### ¿Qué hace? (para el usuario)

Muestra un panel/modal con la lista completa de productos cuyo stock actual está por debajo del umbral definido. Para cada producto indica:
- Nombre e ID del producto
- Tienda/proveedor al que pertenece
- Stock actual
- Umbral mínimo que aplica (puede ser personalizado por producto o el global)
- Un badge de color diferente si el mínimo está personalizado vs. si usa el umbral global

El usuario puede buscar dentro del listado por nombre, ID o tienda.

---

### Decoradores y configuración técnica

```typescript
@Component({
  selector: 'app-bajo-stock-modal',
  standalone: true,
  imports: [CommonModule, FormsModule],
})
export class BajoStockModalComponent
```

---

### Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `@Input() isOpen` | `boolean` | Controla si el modal es visible |
| `@Input() productos` | `ResumenItem[]` | Lista de productos en bajo stock (ya filtrada por el padre) |
| `@Input() umbral` | `number` | Umbral global configurado en el header |
| `@Input() stockMinimoMap` | `Map<string, {_id, StockMinimo}>` | Mínimos personalizados por producto |
| `@Output() close` | `EventEmitter<void>` | Notifica al padre que el usuario cerró el modal |
| `busqueda` | `string` | Texto del buscador interno |

**Getter:**

| Getter | Retorna | Lógica |
|---|---|---|
| `productosFiltrados` | `ResumenItem[]` | Filtra `productos` por `busqueda` (Nombre, IdProducto, Tienda) |

**Métodos:**

| Método | Descripción |
|---|---|
| `getMinimoEfectivo(p)` | Retorna el mínimo real del producto: personalizado si existe, umbral global si no |
| `tieneConfiguracionPropia(p)` | `true` si el producto tiene un StockMinimo > 0 en `stockMinimoMap` |

---

### Flujo principal

```
Padre hace modalBajoStockAbierto = true
  └─► isOpen = true → modal visible

Usuario busca "boxer"
  └─► productosFiltrados recalcula → muestra solo coincidencias

Usuario cierra modal
  └─► close.emit() → padre hace modalBajoStockAbierto = false
```

---

### Historial de cambios

| Fecha | Cambio |
|---|---|
| 2026-05-26 | Creación con soporte de stock mínimo personalizado |

---

### Observaciones

- El componente no filtra los productos por debajo del umbral; eso lo hace el getter `productosBajoStock` en el padre. Este componente solo muestra lo que le llega.
- `getMinimoEfectivo()`: un `StockMinimo` de `0` se trata como "sin configuración" y cae al umbral global. Así se evita confundir "no configurado" con "mínimo cero".
