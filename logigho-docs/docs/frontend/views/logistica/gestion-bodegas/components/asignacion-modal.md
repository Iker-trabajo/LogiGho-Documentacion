## Componente: AsignacionModalComponent

**Selector:** `app-asignacion-modal`  
**Ubicación:** `src/app/views/logistica/gestion-bodegas/components/asignacion-modal/asignacion-modal.component.ts`  
**Acceso:** Se abre desde el botón "Asignar producto" del header (visible solo para roles con permiso `asignar`)

---

### ¿Qué hace? (para el usuario)

Modal para crear una nueva asignación bodega-producto. El usuario selecciona una bodega activa del catálogo, busca y selecciona un producto disponible para sus tiendas, y define el estado inicial de la asignación (Activa o Inactiva).

---

### Decoradores y configuración técnica

```typescript
@Component({
  selector: 'app-asignacion-modal',
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule, FormsModule],
})
export class AsignacionModalComponent
```

---

### Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `@Input() isOpen` | `boolean` | Controla visibilidad |
| `@Input() bodegas` | `Bodega[]` | Solo muestra bodegas con `estado === 'Activa'` |
| `@Input() productos` | `ProductoOpcion[]` | Catálogo de productos disponibles para las tiendas del usuario |
| `@Input() guardando` | `boolean` | Deshabilita el botón mientras guarda |
| `@Output() guardar` | `EventEmitter<{idBodega, idProducto, estado}>` | Payload de la nueva asignación |
| `@Output() cerrar` | `EventEmitter<void>` | Cierra el modal |

---

### Flujo principal

```
Padre carga productos (una sola vez, con caché)
  └─► abrirModalAsignacion() → asignacionModalAbierta = true

Usuario selecciona bodega, producto y estado
  └─► guardar.emit({ idBodega, idProducto, estado })
        └─► padre llama orchestrator.crearAsignacion()
              └─► ok → cerrarModalAsignacion()

Usuario cierra
  └─► cerrar.emit() → padre pone asignacionModalAbierta = false
```

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-06-24 | Adalberto González | Creación con búsqueda de productos y selector de bodegas activas |

---

### Observaciones

- El catálogo de productos se carga filtrado por las tiendas asignadas al usuario (`sessionStorage['tiendas_asignadas']`), a diferencia del stock de bodegas que es global.
