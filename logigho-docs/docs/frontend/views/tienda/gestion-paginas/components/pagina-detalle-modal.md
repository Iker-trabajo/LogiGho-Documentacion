## Componente: PaginaDetalleModalComponent

**Selector:** `app-pagina-detalle-modal`  
**Ubicación:** `src/app/views/tienda/gestion-paginas/components/pagina-detalle-modal/pagina-detalle-modal.component.ts`  
**Acceso:** Se abre al hacer clic en una fila de la tabla, o en una tarjeta KPI

---

### ¿Qué hace?

Modal genérico de dos modos:

1. **Modo `usuarios`**: al hacer clic en una fila de la tabla, muestra el listado de usuarios asignados a esa página (nombre, teléfono, cuenta de Facebook vinculada y estado Activo/Removido). Incluye buscador interno por nombre.
2. **Modo `paginas`**: al hacer clic en una tarjeta KPI (ej. "Páginas activas"), muestra el listado de páginas que cumplen ese criterio; al elegir una, abre su detalle de usuarios.

Ambos modos comparten el mismo buscador y el mismo estilo de lista (`.upm-list` / `.upm-item`), solo cambia qué información se muestra en cada fila.

---

### Decoradores y configuración técnica

```typescript
@Component({
  selector: 'app-pagina-detalle-modal',
  standalone: true,
  imports: [CommonModule, FormsModule],
  templateUrl: './pagina-detalle-modal.component.html',
  styleUrl: './pagina-detalle-modal.component.scss',
})
export class PaginaDetalleModalComponent

export type PaginaDetalleModalModo = 'usuarios' | 'paginas';
```

> Reemplaza al antiguo `UsuariosPaginaModalComponent`, que solo tenía el modo "usuarios". Se unificó en un solo componente de dos modos siguiendo el mismo patrón que `KpiBodegasModalComponent` de `gestion-bodegas`.

---

### Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `@Input() isOpen` | `boolean` | Controla visibilidad del modal |
| `@Input() modo` | `PaginaDetalleModalModo` | `'usuarios'` o `'paginas'` |
| `@Input() pagina` | `Pagina \| null` | Página cuyo detalle de usuarios se muestra (modo `usuarios`) |
| `@Input() titulo` / `@Input() subtitulo` | `string` | Título y subtítulo del modal (modo `paginas`) |
| `@Input() paginas` | `Pagina[]` | Páginas que cumplen el criterio del KPI clickeado (modo `paginas`) |
| `@Output() cerrar` | `EventEmitter<void>` | Cierra el modal |
| `@Output() verPagina` | `EventEmitter<Pagina>` | Emitido al elegir una página dentro del listado (modo `paginas`) |
| `busqueda` | `string` | Texto de búsqueda interna, compartido por ambos modos |
| `usuarios` / `usuariosFiltrados` | getter | Usuarios de `pagina`, filtrados por `busqueda` |
| `paginasFiltradas` | getter | `paginas` filtradas por `busqueda` |

---

### Flujo principal

```
Modo 'usuarios' — clic en una fila de la tabla
  └─► abrirModalUsuarios(pagina) → modalModo = 'usuarios', paginaEnModal = pagina
        └─► Muestra usuarios de pagina.usuarios (activos y removidos, diferenciados)

Modo 'paginas' — clic en una tarjeta KPI
  └─► abrirKpiDetalle(tipo) → modalModo = 'paginas', modalKpiPaginas = [...]
        └─► Usuario elige una página de la lista
              └─► seleccionarPagina(p) → verPagina.emit(p) + cerrar.emit()
                    └─► El padre vuelve a abrir el modal, ahora en modo 'usuarios', para esa página

Usuario cierra (Escape, clic en fondo oscuro o botón cerrar)
  └─► cerrar.emit() → el padre pone modalAbierto = false
```

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-16 | Adalberto González | Creación del modal de detalle de usuarios (`UsuariosPaginaModalComponent`), reemplazando el diseño inicial de fila expandible/sub-tabla dentro de la tabla maestro |
| 2026-07-24 | Adalberto González | Se unifica con un segundo modo (`'paginas'`) para el detalle por KPI, renombrando el componente a `PaginaDetalleModalComponent` |

---

### Observaciones

- Los usuarios removidos se muestran diferenciados (opacidad reducida, badge distinto), nunca ocultos — es el histórico de quién tuvo acceso a esa página.
- No recibe ni expone `tokenAccesoPagina` en ningún punto — ese campo ya se filtra en `GestionPaginasRepository` antes de llegar a cualquier componente de presentación.
- El buscador (`.upm-search`) y la lista (`.upm-list` / `.upm-item`) de este modal son el mismo patrón visual reutilizado en `ProductoSelectorModalComponent` para mostrar los productos ya asociados a una página.
