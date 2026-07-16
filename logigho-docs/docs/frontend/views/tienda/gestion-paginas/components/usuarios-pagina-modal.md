## Componente: UsuariosPaginaModalComponent

**Selector:** `app-usuarios-pagina-modal`  
**Ubicación:** `src/app/views/tienda/gestion-paginas/components/usuarios-pagina-modal/usuarios-pagina-modal.component.ts`  
**Acceso:** Se abre al hacer clic en cualquier fila de la tabla de páginas

---

### ¿Qué hace? (para el usuario)

Modal que muestra el listado de usuarios asignados a la página seleccionada: nombre, teléfono, cuenta de Facebook vinculada y estado (`Activo` / `Removido`). Incluye buscador interno por nombre. Los usuarios removidos se muestran diferenciados (opacidad reducida, badge distinto), nunca ocultos — es el histórico de quién tuvo acceso a esa página. Si la página no tiene usuarios asignados, muestra un estado vacío explicativo. Cierra con tecla Escape o clic en el fondo oscuro.

---

### Decoradores y configuración técnica

```typescript
@Component({
  selector: 'app-usuarios-pagina-modal',
  standalone: true,
  imports: [CommonModule, FormsModule],
  templateUrl: './usuarios-pagina-modal.component.html',
  styleUrl: './usuarios-pagina-modal.component.scss',
})
export class UsuariosPaginaModalComponent
```

> Modelado directamente sobre el patrón visual de `KpiBodegasModalComponent` de `gestion-bodegas`: mismo `position: fixed` con `z-index` alto (20001 backdrop / 20002 modal), mismo buscador interno y misma estructura de header/body/footer. Es una copia del patrón, no una dependencia compartida — no hay import cruzado entre los módulos de `logistica` y `tienda`.

---

### Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `@Input() isOpen` | `boolean` | Controla visibilidad del modal |
| `@Input() pagina` | `Pagina \| null` | Página cuyo detalle de usuarios se muestra |
| `@Output() cerrar` | `EventEmitter<void>` | Cierra el modal |
| `busqueda` | `string` | Texto de búsqueda interna por nombre de usuario |
| `usuarios` | getter `Usuario[]` | Atajo a `pagina?.usuarios ?? []` |
| `usuariosFiltrados` | getter `Usuario[]` | `usuarios` filtrado por `busqueda` (case-insensitive, sobre `nombre`) |

---

### Flujo principal

```
Padre abre el modal (modalAbierto = true, paginaEnModal = pagina)
  └─► Renderiza usuarios de pagina.usuarios (activos y removidos, diferenciados)

Usuario escribe en el buscador interno
  └─► usuariosFiltrados se recalcula en cada tecla

Usuario cierra (Escape, clic en backdrop o botón cerrar)
  └─► cerrar.emit() → padre pone modalAbierto = false, paginaEnModal = null
```

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-16 | Adalberto González | Creación del modal de detalle de usuarios, reemplazando el diseño inicial de fila expandible/sub-tabla dentro de la tabla maestro |

---

### Observaciones

- Este componente reemplazó un diseño anterior donde cada fila de la tabla se podía expandir para mostrar una sub-tabla de usuarios inline. Se cambió a modal para unificar la experiencia con el patrón ya validado en `gestion-bodegas` (clic en KPI/fila → modal), y porque la sub-tabla inline no tenía paginación ni espacio suficiente para mostrarse bien dentro de la fila.
- No recibe ni expone `tokenAccesoPagina` ni `_id` en ningún punto — esos campos ya se filtran en `GestionPaginasRepository` antes de llegar a cualquier componente de presentación.
