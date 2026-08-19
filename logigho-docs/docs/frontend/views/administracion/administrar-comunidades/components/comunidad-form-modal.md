## Componente: ComunidadFormModalComponent

**Selector:** `app-comunidad-form-modal`  
**Ubicación:** `src/app/views/administracion/administrar-comunidades/components/comunidad-form-modal/comunidad-form-modal.component.ts`  
**Acceso:** Se abre desde los botones "Nueva comunidad" y "Editar" de `AdministrarComunidadesComponent`

---

### ¿Qué hace? (para el usuario)

Modal con formulario reactivo para crear o editar una comunidad: tienda dueña (solo `Proveedor`, deshabilitada en modo edición), nombre, descripción y administradores. Los selectores de tienda dueña y de administradores son tipo buscador + dropdown, mismo lenguaje visual que `filtros-inventario`.

---

### Decoradores y configuración técnica

```typescript
@Component({
  selector: 'app-comunidad-form-modal',
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule, FormsModule],
  templateUrl: './comunidad-form-modal.component.html',
  styleUrl: './comunidad-form-modal.component.scss',
})
export class ComunidadFormModalComponent
```

---

### Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `@Input() isOpen` | `boolean` | Controla visibilidad |
| `@Input() form` | `FormGroup` | Formulario preconstruido por el padre |
| `@Input() formMode` | `FormMode` | `'nueva'` o `'edicion'` — define título y si la tienda dueña es editable |
| `@Input() tiendas` | `{id, nombre}[]` | Tiendas `Proveedor` disponibles como dueña |
| `@Input() usuarios` | `{email, username}[]` | Universo de administradores elegibles |
| `@Output() guardar` | `EventEmitter<void>` | El padre valida y llama al repository |
| `@Output() cerrar` | `EventEmitter<void>` | Cierra el modal |

---

### Flujo principal

```
Padre abre modal
  ├─► modo 'nueva'    → form vacío, tienda dueña habilitada
  └─► modo 'edicion'  → form precargado, tienda dueña deshabilitada, checkbox
                         "crear comunidad social" deshabilitado (solo aplica al crear)

Usuario busca y selecciona tienda dueña / administradores
  └─► seleccionarTienda() / toggleAdmin() actualizan el FormGroup del padre

Usuario guarda
  └─► guardar.emit() → padre valida y llama a crearComunidad()/editarComunidad()

Escape / clic fuera
  └─► cierra primero cualquier dropdown abierto; si ninguno, cierra el modal
```

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-08-19 | Adalberto González | Documentación inicial, ya en producción |

---

### Observaciones

- El dropdown de administradores solo se pinta a partir de que el usuario empieza a escribir (`searchAdmin` no vacío) — con la lista completa de usuarios visible desde el foco, tapaba el campo siguiente del formulario.
- Mismo patrón visual que `CuentaFormModalComponent` (backdrop con blur, header azul, footer con botones).
