## Componente: CuentaFormModalComponent

**Selector:** `app-cuenta-form-modal`  
**Ubicación:** `src/app/views/tienda/gestion-cuentas/components/cuenta-form-modal/cuenta-form-modal.component.ts`  
**Acceso:** Se abre desde los botones "Nueva cuenta" y "Editar" de la tabla

---

### ¿Qué hace? (para el usuario)

Modal con un formulario reactivo para crear o editar una cuenta. El usuario ingresa nombre, token de acceso (obtenido manualmente del panel de Meta) y estado. El título cambia dinámicamente entre "Nueva cuenta" y "Editar cuenta" según el contexto. Cierra con tecla Escape.


---

### Decoradores y configuración técnica

```typescript
@Component({
  selector: 'app-cuenta-form-modal',
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule],
})
export class CuentaFormModalComponent
```

---

### Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `@Input() isOpen` | `boolean` | Controla visibilidad del modal |
| `@Input() form` | `FormGroup` | Formulario reactivo preconstruido por el componente padre (`nombre`, `tokenAcceso`, `estado`) |
| `@Input() formMode` | `FormMode` | `'nueva'` o `'edicion'`, define el título mostrado |
| `@Input() guardando` | `boolean` | Deshabilita el botón Guardar mientras espera respuesta |
| `@Output() guardar` | `EventEmitter<void>` | El padre valida y llama al repositorio |
| `@Output() cerrar` | `EventEmitter<void>` | Cierra el modal |

---

### Flujo principal

```
Padre abre modal (showForm = true)
  ├─► Modo creación: form en blanco
  └─► Modo edición:  form precargado con datos de la cuenta

Usuario completa el formulario y hace clic en Guardar
  └─► guardar.emit() → padre valida el FormGroup
        ├─► Válido   → repo.crearCuenta() o editarCuenta()
        └─► Inválido → marca errores visualmente (getError / esCampoInvalido)

Usuario cierra (Escape o botón cerrar)
  └─► cerrar.emit() → padre pone showForm = false
```

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-15 | Adalberto González | Creación del modal con formulario reactivo y soporte de edición, extraído como componente propio de `GestionCuentasComponent` |

---

### Observaciones

- El formulario es construido y validado en el padre (`GestionCuentasComponent.buildForm()`). Este componente solo lo renderiza y emite eventos, igual que `BodegaFormModalComponent`.
- Header azul `var(--logigho-azul)`, con `z-index` intencionalmente alto (20001 backdrop / 20002 modal) para garantizar que se muestre por encima del layout de la aplicación (sidebar, header) independientemente de dónde se monte el componente en el árbol.
