## Componente: BodegaFormModalComponent

**Selector:** `app-bodega-form-modal`  
**Ubicación:** `src/app/views/logistica/gestion-bodegas/components/bodega-form-modal/bodega-form-modal.component.ts`  
**Acceso:** Se abre desde los botones "Nueva bodega" y "Editar" de la tabla

---

### ¿Qué hace? (para el usuario)

Modal con un formulario reactivo para crear o editar una bodega. El usuario puede ingresar nombre, ubicación, ciudad, encargado y estado. El título cambia dinámicamente entre "Nueva bodega" y "Editar bodega" según el contexto. Cierra con tecla Escape o clic fuera del modal.

---

### Decoradores y configuración técnica

```typescript
@Component({
  selector: 'app-bodega-form-modal',
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule],
})
export class BodegaFormModalComponent
```

---

### Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `@Input() isOpen` | `boolean` | Controla visibilidad del modal |
| `@Input() form` | `FormGroup` | Formulario reactivo preconstruido por `BodegaFormService` en el padre |
| `@Input() titulo` | `string` | "Nueva bodega" o "Editar bodega" |
| `@Input() guardando` | `boolean` | Deshabilita el botón Guardar mientras espera respuesta |
| `@Input() ciudades` | `Ciudad[]` | Lista para el selector de ciudad |
| `@Output() guardar` | `EventEmitter<void>` | El padre valida y llama al orquestador |
| `@Output() cerrar` | `EventEmitter<void>` | Cierra el modal |

---

### Flujo principal

```
Padre abre modal (modalAbierto = true)
  ├─► Modo creación: form en blanco
  └─► Modo edición:  form precargado con datos de la bodega

Usuario completa el formulario y hace clic en Guardar
  └─► guardar.emit() → padre valida con BodegaFormService
        ├─► Válido   → orchestrator.crearBodega() o editarBodega()
        └─► Inválido → marca errores visualmente

Usuario cierra
  └─► cerrar.emit() → padre pone modalAbierto = false
```

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-06-24 | Adalberto González | Creación del modal con formulario reactivo y soporte de edición |

---

### Observaciones

- El formulario es construido y validado en el padre (`BodegaFormService`). Este componente solo lo renderiza y emite eventos.
- Header azul `var(--logigho-azul)`, footer con fondo `var(--logigho-blanco-hueso)`.
