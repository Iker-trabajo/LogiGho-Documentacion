## Componente: AsignacionesDrawerComponent

**Selector:** `app-asignaciones-drawer`  
**Ubicación:** `src/app/views/logistica/gestion-bodegas/components/asignaciones-drawer/asignaciones-drawer.component.ts`  
**Acceso:** Se abre desde el botón "Gestión" de cada fila de la tabla (visible para roles con permiso `gestionAsignaciones`)

---

### ¿Qué hace? (para el usuario)

Modal que muestra todos los productos asignados a una bodega específica. El usuario puede buscar por ID o nombre de producto, activar o desactivar una asignación individualmente, y eliminarla con una confirmación inline. Las acciones de editar y eliminar solo aparecen si el usuario tiene el permiso correspondiente.

---

### Decoradores y configuración técnica

```typescript
@Component({
  selector: 'app-asignaciones-drawer',
  standalone: true,
  imports: [CommonModule, FormsModule],
})
export class AsignacionesDrawerComponent implements OnChanges
```

---

### Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `@Input() isOpen` | `boolean` | Controla visibilidad |
| `@Input() bodega` | `Bodega \| null` | Bodega cuyos productos se gestionan |
| `@Input() asignaciones` | `AsignacionBodega[]` | Lista completa de asignaciones de la bodega |
| `@Input() cargando` | `boolean` | Muestra skeleton mientras carga |
| `@Input() nombresProducto` | `Map<string, string>` | `idProducto → nombre` para mostrar junto al ID |
| `@Input() puedeEditarAsignacion` | `boolean` | Muestra botón de toggle de estado |
| `@Input() puedeEliminarAsignacion` | `boolean` | Muestra botón de eliminar |
| `@Output() cerrar` | `EventEmitter<void>` | Cierra el modal |
| `@Output() toggleAsignacion` | `EventEmitter<AsignacionBodega>` | Solicita cambio de estado al padre |
| `@Output() eliminarAsignacion` | `EventEmitter<AsignacionBodega>` | Solicita eliminación al padre |
| `busqueda` | `string` | Texto del filtro interno de búsqueda |
| `confirmandoEliminar` | `string \| null` | `_id` de la asignación en proceso de confirmación de borrado |

**Getter:**

| Getter | Descripción |
|---|---|
| `asignacionesFiltradas` | Filtra `asignaciones` por `idProducto` o nombre de producto según `busqueda` |

---

### Flujo principal

```
Padre abre el modal (drawerAbierto = true, bodegaDrawer = bodega)
  └─► ngOnChanges resetea busqueda = '' y confirmandoEliminar = null

Usuario busca "camisa"
  └─► asignacionesFiltradas recalcula → muestra solo coincidencias

Usuario hace clic en eliminar una asignación
  └─► confirmandoEliminar = asignacion._id → botones de confirmación inline visibles
        ├─► Confirma → eliminarAsignacion.emit() → padre llama orquestador
        └─► Cancela  → confirmandoEliminar = null

Usuario cierra
  └─► cerrar.emit() → padre pone drawerAbierto = false
```

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-06-24 | Adalberto González | Creación con búsqueda, confirmación inline de eliminación y control por rol |

---

### Observaciones

- Cierra con tecla Escape (`@HostListener('document:keydown.escape')`) y clic en el overlay.
- `ngOnChanges` resetea el estado interno al cambiar de bodega para evitar que la búsqueda de una bodega anterior quede visible al abrir otra.
- El footer muestra "N de Total asignaciones" solo cuando hay una búsqueda activa.
