## Componente: BodegaCardComponent

**Selector:** `app-bodega-card`  
**Ubicación:** `src/app/views/logistica/gestion-bodegas/components/bodega-card/bodega-card.component.ts`  
**Acceso:** Usado internamente como fila `<tr>` de la tabla de bodegas

---

### ¿Qué hace? (para el usuario)

Renderiza una fila de la tabla con el nombre, ID, ubicación, encargado, stock actual y estado de una bodega. Si el usuario tiene permisos, muestra botones para editar, deshabilitar/habilitar y gestionar asignaciones de productos.

---

### Decoradores y configuración técnica

```typescript
@Component({
  selector: 'app-bodega-card',
  standalone: true,
  imports: [CommonModule],
})
export class BodegaCardComponent
```

Componente dumb: solo recibe datos por `@Input()` y emite eventos por `@Output()`. No tiene lógica de negocio ni llama al backend.

---

### Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `@Input() bodega` | `Bodega` | Datos de la bodega a renderizar |
| `@Input() stock` | `number` | Stock total calculado para esta bodega desde `stockPorBodega` |
| `@Input() puedeEditar` | `boolean` | Muestra botón Editar |
| `@Input() puedeToggle` | `boolean` | Muestra botón Deshabilitar/Habilitar |
| `@Input() puedeGestion` | `boolean` | Muestra botón Gestión (asignaciones) |
| `@Input() mostrarAcciones` | `boolean` | Muestra la celda de acciones completa. `false` oculta toda la columna |
| `@Output() editar` | `EventEmitter<Bodega>` | Emite al hacer clic en Editar |
| `@Output() toggleEstado` | `EventEmitter<Bodega>` | Emite al hacer clic en Deshabilitar/Habilitar |
| `@Output() verAsignaciones` | `EventEmitter<Bodega>` | Emite al hacer clic en Gestión |

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-06-24 | Adalberto González | Creación con soporte de permisos por rol, columna de stock e inputs de acciones |

---

### Observaciones

- El componente no conoce el sistema de roles. El padre le pasa `true/false` por cada permiso y el componente solo muestra u oculta.
- `mostrarAcciones = false` oculta la celda `<td>` completa, lo que colapsa la columna Acciones para usuarios sin ningún permiso.
