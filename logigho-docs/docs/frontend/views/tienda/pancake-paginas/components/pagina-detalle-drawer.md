## Componente: PaginaDetalleDrawerComponent

**Selector:** `app-pagina-detalle-drawer`  
**Ubicación:** `src/app/views/tienda/pancake-paginas/components/pagina-detalle-drawer/pagina-detalle-drawer.component.ts`  
**Acceso:** Se abre al hacer clic en el botón de "ver detalle" (lupa) de una fila de la tabla

---

### ¿Qué hace?

Cajón lateral con el detalle de una página, organizado en 3 pestañas:

1. **Campañas**: separadas en activas y pausadas, cada una con barra de progreso de presupuesto, gasto y alcance.
2. **Usuarios**: separados en activos y removidos (estos últimos diferenciados, nunca ocultos).
3. **Productos**: lista de productos asociados a esa página, configurados desde `gestion-paginas`. Si no tiene ninguno, muestra un estado vacío.

No hace llamadas al backend — todo lo que muestra llega ya calculado desde el componente padre (`PancakePaginasComponent`).

---

### Decoradores y configuración técnica

```typescript
@Component({
  selector: 'app-pagina-detalle-drawer',
  standalone: true,
  imports: [CommonModule],
  templateUrl: './pagina-detalle-drawer.component.html',
  styleUrl: './pagina-detalle-drawer.component.scss'
})
export class PaginaDetalleDrawerComponent implements OnChanges
```

---

### Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `@Input() isOpen` | `boolean` | Controla si el cajón está abierto o cerrado |
| `@Input() pagina` | `PaginaResumen \| null` | Página cuyo detalle se muestra (usuarios y productos salen de aquí) |
| `@Input() campanas` | `CampanaResumen[]` | Campañas ya resueltas de esa página (calculadas por el padre) |
| `@Output() cerrar` | `EventEmitter<void>` | Cierra el cajón |
| `tabActiva` | `'campanas' \| 'usuarios' \| 'productos'` | Pestaña actualmente visible |
| `usuarios` / `usuariosActivos` / `usuariosRemovidos` | getters | Atajos sobre `pagina.usuarios`, ya separados por estado |
| `productosAsociados` | getter `ProductoAsociado[]` | Atajo a `pagina?.productosAsociados ?? []` |
| `campanasActivas` / `campanasPausadas` | getters | Atajos sobre `campanas`, separadas por estado |

---

### Flujo principal

```
Padre abre el drawer (drawerAbierto = true, _paginaSeleccionada.set(pagina))
  └─► ngOnChanges detecta un nuevo `pagina`
        └─► tabActiva vuelve a 'campanas' (siempre empieza en la primera pestaña)

Usuario hace clic en una pestaña
  └─► cambiarTab(tab) → tabActiva = tab
        └─► Se muestra la sección correspondiente (campañas, usuarios o productos)

Usuario cierra (Escape, clic en fondo oscuro o botón cerrar)
  └─► cerrar.emit() → el padre pone drawerAbierto = false
```

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-24 | Adalberto González | Documentación inicial. Se agregó la pestaña "Productos" para mostrar los productos asociados a la página, guardados desde el módulo `gestion-paginas` |

---

### Observaciones

- Es un componente puramente de presentación: no tiene inyección de servicios ni hace HTTP. Todo lo que necesita llega por `@Input()`.
- La pestaña "Productos" solo muestra el nombre de cada producto — no muestra imagen ni SKU, para mantener la lista simple.
- Este drawer y el modal de selección de productos (`ProductoSelectorModalComponent`, en `gestion-paginas`) muestran la misma información (`productosAsociados`) pero uno es de solo lectura (drawer) y el otro permite editarla (modal).
