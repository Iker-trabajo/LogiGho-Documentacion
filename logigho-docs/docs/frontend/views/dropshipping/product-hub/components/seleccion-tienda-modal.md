## Componente: SeleccionTiendaModalComponent

**Selector:** `app-seleccion-tienda-modal`  
**Ubicación:** `src/app/views/dropshipping/product-hub/explorar-comunidades/components/seleccion-tienda-modal/seleccion-tienda-modal.component.ts`  
**Acceso:** Usado por `ExplorarComunidadesComponent` y `ComunidadDetalleComponent` cuando el usuario tiene varias tiendas elegibles

---

### ¿Qué hace? (para el usuario)

Modal simple que pide elegir a nombre de cuál tienda actuar, cuando el usuario tiene más de una tienda `Propia`/`Dropshipping` elegible para solicitar membresía o permiso de producto. Incluye buscador por nombre. Cierra con Escape o clic en el backdrop.

---

### Decoradores y configuración técnica

```typescript
@Component({
  selector: 'app-seleccion-tienda-modal',
  imports: [CommonModule, FormsModule],
  templateUrl: './seleccion-tienda-modal.component.html',
  styleUrl: './seleccion-tienda-modal.component.scss',
})
export class SeleccionTiendaModalComponent implements OnChanges
```

---

### Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `@Input() isOpen` | `boolean` | Controla visibilidad del modal |
| `@Input() tiendas` | `{id, nombre}[]` | Tiendas elegibles entre las que puede elegir el usuario |
| `@Output() elegir` | `EventEmitter<{id, nombre}>` | Notifica la tienda elegida |
| `@Output() cerrar` | `EventEmitter<void>` | Notifica el cierre sin elegir |

---

### Flujo principal

```
Padre abre el modal con la lista de tiendas elegibles
  └─► ngOnChanges reinicia selección y búsqueda

Usuario busca, selecciona una tienda y confirma
  └─► confirmar() → elegir.emit(tienda)

Usuario cierra (Escape / backdrop)
  └─► cerrar.emit()
```

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-08-19 | Adalberto González | Documentación inicial, ya en producción |

---

### Observaciones

- Componente puro: no llama al backend, solo emite la tienda elegida por `@Output()`.
