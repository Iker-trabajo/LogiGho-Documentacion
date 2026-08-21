## Componente: SolicitudModalComponent

**Selector:** `app-solicitud-modal`  
**Ubicación:** `src/app/views/dropshipping/product-hub/gestion-comunidad/components/solicitud-modal/solicitud-modal.component.ts`  
**Acceso:** Reutilizado en 3 flujos: crear solicitud de membresía, crear solicitud de producto, y resolver (aprobar/rechazar) ambos tipos en `gestion-comunidad`

---

### ¿Qué hace? (para el usuario)

Modal genérico con un formulario de motivo obligatorio. Tiene dos modos:

- **Crear** (`soloCrear: true`): usado por `explorar-comunidades` y `comunidad-detalle` para capturar el motivo al solicitar unirse o pedir permiso de un producto.
- **Resolver** (por defecto): usado por `gestion-comunidad` para que el admin elija Aprobar/Rechazar con motivo, sobre una solicitud de membresía o de producto ya existente.

---

### Decoradores y configuración técnica

```typescript
@Component({
  selector: 'app-solicitud-modal',
  imports: [CommonModule, ReactiveFormsModule],
  templateUrl: './solicitud-modal.component.html',
  styleUrl: './solicitud-modal.component.scss',
})
export class SolicitudModalComponent implements OnChanges
```

---

### Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `@Input() isOpen` | `boolean` | Controla visibilidad |
| `@Input() titulo` | `string` | Título del modal |
| `@Input() nombreSolicitante` | `string` | Usuario/tienda que originó la solicitud |
| `@Input() detalleExtra` | `string \| null` | Detalle adicional (ej. nombre del producto) |
| `@Input() soloCrear` | `boolean` | `true` para el flujo de creación (oculta el selector Aprobar/Rechazar) |
| `@Output() resolver` | `EventEmitter<ResolucionSolicitud>` | `{ accion: 'APROBAR'\|'RECHAZAR', motivo: string }` |
| `@Output() cerrar` | `EventEmitter<void>` | Cierra sin enviar |

---

### Flujo principal

```
Padre abre el modal (isOpen = true)
  └─► ngOnChanges reinicia el form { accion: 'APROBAR', motivo: '' }

Usuario completa motivo y envía
  └─► enviar() valida el FormGroup
        ├─► inválido → markAllAsTouched, no emite
        └─► válido   → resolver.emit({ accion, motivo })
```

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-08-19 | Adalberto González | Documentación inicial, ya en producción |

---

### Observaciones

- El mismo componente sirve para crear y para resolver solicitudes — el padre decide qué hacer con el evento `resolver` según el contexto en que lo abrió.
