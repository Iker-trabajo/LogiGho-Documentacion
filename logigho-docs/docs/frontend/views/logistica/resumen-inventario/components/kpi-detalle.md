
## 7. Componente: KpiDetalleModalComponent

**Selector:** `app-kpi-detalle-modal`
**Ubicación:** `src/app/views/logistica/resumen-inventario/components/kpi-detalle-modal/kpi-detalle-modal.component.ts`
**Acceso:** Se abre al hacer click en las tarjetas KPI clicables

---

### ¿Qué hace? (para el usuario)

Muestra un modal con la lista de productos que corresponden al KPI seleccionado (por ejemplo: "Productos Activos" o "Bajo Stock"). Para cada producto muestra:
- Imagen del producto (si está disponible)
- Nombre, variación, color e ID
- Tienda/proveedor
- Stock actual

El usuario puede buscar dentro del listado por nombre, ID o tienda.

---

### Decoradores y configuración técnica

```typescript
@Component({
  selector: 'app-kpi-detalle-modal',
  standalone: true,
  imports: [CommonModule, FormsModule],
})
export class KpiDetalleModalComponent
```

---

### Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `@Input() isOpen` | `boolean` | Controla si el modal es visible |
| `@Input() titulo` | `string` | Título del modal (ej: "Productos Activos") |
| `@Input() productos` | `ResumenItem[]` | Lista de productos a mostrar |
| `@Input() imagenMap` | `Map<string, string>` | IdProducto → URL de imagen |
| `@Output() close` | `EventEmitter<void>` | Notifica al padre que el usuario cerró el modal |

**Getter:**

| Getter | Descripción |
|---|---|
| `productosFiltrados` | Filtra `productos` por búsqueda en Nombre, IdProducto, Tienda |

**Métodos:**

| Método | Descripción |
|---|---|
| `getImagen(p)` | Busca la URL de imagen en `imagenMap`; retorna string vacío si no existe |

---

### Flujo principal

```
Padre llama abrirKpiDetalle(titulo, productos)
  └─► modalKpiTitulo y modalKpiDatos se actualizan
      modalKpiAbierto = true → isOpen = true → modal visible

Usuario busca dentro del modal
  └─► productosFiltrados recalcula en tiempo real

Usuario cierra el modal
  └─► close.emit() → padre pone modalKpiAbierto = false
```

---

### Historial de cambios

| Fecha | Cambio |
|---|---|
| 2026-05-26 | Creación con soporte de imagen por producto |

---

### Observaciones

- `getImagen()` retorna `''` en lugar de `undefined` para que Angular no lance error en el binding `[src]=""`. El `*ngIf` en el HTML oculta la imagen si el string está vacío.
