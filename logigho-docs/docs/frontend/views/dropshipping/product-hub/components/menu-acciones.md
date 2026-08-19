## Componente: MenuAccionesComponent

**Selector:** `app-menu-acciones`  
**Ubicación:** `src/app/components/menu-acciones/menu-acciones.component.ts`  
**Acceso:** Componente compartido del proyecto (`src/app/components/`), no vive dentro de `product-hub`. Se documenta aquí porque su primer y único consumidor actual es `ComunidadDetalleComponent`.

---

### ¿Qué hace? (para el usuario)

Botón circular de "3 puntos" (estilo Facebook) que despliega un menú de acciones secundarias. En `comunidad-detalle` agrupa "Gestionar comunidad" y "Salir de la comunidad" fuera del flujo principal del banner. Pensado para ser reutilizado en cualquier otro banner del proyecto que necesite el mismo patrón.

---

### Decoradores y configuración técnica

```typescript
@Component({
  selector: 'app-menu-acciones',
  standalone: true,
  imports: [CommonModule],
  templateUrl: './menu-acciones.component.html',
  styleUrl: './menu-acciones.component.scss',
})
export class MenuAccionesComponent
```

---

### Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `@Input() opciones` | `OpcionMenu[]` | `{ id, etiqueta, variante?: 'default'\|'peligro' }` — el consumidor decide qué opciones mostrar |
| `@Output() seleccionar` | `EventEmitter<string>` | Emite el `id` de la opción elegida |
| `abierto` | `boolean` | Estado de visibilidad del dropdown |

---

### Flujo principal

```
Usuario hace clic en el botón de 3 puntos
  └─► toggle() → abierto = !abierto

Usuario elige una opción
  └─► elegir(id) → abierto = false → seleccionar.emit(id)

Click fuera del componente o Escape
  └─► onClickFuera() / onEscape() → abierto = false
```

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-08-19 | Adalberto González | Documentación inicial, ya en producción |

---

### Observaciones

- Componente puro sin lógica de negocio: no sabe qué significa cada `id`, el consumidor lo despacha (ver `onAccionMenu` en `ComunidadDetalleComponent`).
- Usa `ElementRef` + `HostListener('document:click')` para detectar clics fuera, mismo patrón que otros dropdowns del proyecto.
