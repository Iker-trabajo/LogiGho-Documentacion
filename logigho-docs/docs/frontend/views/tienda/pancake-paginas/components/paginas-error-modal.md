## Componente: PaginasErrorModalComponent

**Selector:** `app-paginas-error-modal`
**Ubicación:** `src/app/views/tienda/pancake-paginas/components/paginas-error-modal/paginas-error-modal.component.ts`
**Acceso:** Se abre al hacer clic en la tarjeta KPI "Páginas con error"

---

### ¿Qué hace?

Modal de listado que muestra todas las páginas con algún error, para que el usuario no tenga que revisar fila por fila la tabla principal buscando el badge de "Error". Cubre los dos tipos de error del módulo:

- **Error de conexión** (`codigoError`): problema a nivel de la página misma con Meta/Pancake — incluye, entre otras causas, un **token de acceso vencido**.
- **Error de sincronización** (`errorSincronizacion`): la última fila de estadísticas de esa página llegó con `status: 'error'` (ver [EstadisticasPipeline](../helpers/estadisticas-pipeline.md)).

Cada fila muestra el nombre de la página, su teléfono formateado, la descripción del error y un badge indicando de qué tipo es. Tiene buscador por nombre. Al hacer clic en una página, solo abre su drawer de detalle (`PaginaDetalleDrawerComponent`) — **no** enfoca la página como cross-filtro del resto del dashboard, para no filtrar la tabla ni los KPIs de fondo al usuario que solo quiere revisar el error.

---

### Decoradores y configuración técnica

```typescript
@Component({
  selector: 'app-paginas-error-modal',
  standalone: true,
  imports: [CommonModule, FormsModule],
  templateUrl: './paginas-error-modal.component.html',
  styleUrl: './paginas-error-modal.component.scss',
})
export class PaginasErrorModalComponent
```

---

### Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `@Input() isOpen` | `boolean` | Controla si el modal está abierto o cerrado |
| `@Input() paginas` | `PaginaResumen[]` | Páginas con error, ya filtradas por el componente padre (`paginasConError` en `PancakePaginasComponent`) |
| `@Output() cerrar` | `EventEmitter<void>` | Cierra el modal |
| `@Output() verPagina` | `EventEmitter<PaginaResumen>` | Se dispara al elegir una página de la lista |
| `busqueda` | `string` | Texto del buscador interno |
| `paginasFiltradas` | getter `PaginaResumen[]` | `paginas` filtradas por `busqueda` (solo por nombre) |
| `descripcionError(p)` | método | Texto a mostrar: prioriza `codigoError` sobre `errorSincronizacion` si la página tiene ambos |
| `formatearTelefono(telefono)` | método | Mismo formato que el resto del módulo: `573233954555` → `+57 323 3954555` |

---

### Flujo principal

```
Usuario hace clic en la tarjeta KPI "Páginas con error"
  └─► PancakePaginasComponent.abrirModalError() → modalErrorAbierto = true

Usuario busca por nombre
  └─► busqueda se actualiza vía [(ngModel)] → paginasFiltradas se recalcula

Usuario hace clic en una página de la lista
  └─► seleccionarPagina(p) → verPagina.emit(p) y cerrar.emit()
        └─► El padre (verPaginaConError) solo llama a abrirDrawer(pagina)
              → NO se enfoca la página ni se filtra el resto del dashboard

Usuario cierra (Escape, clic en fondo oscuro o botón cerrar)
  └─► cerrar.emit() → el padre pone modalErrorAbierto = false
```

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-30 | Adalberto González | Creación del componente. De paso se corrigió el KPI "Páginas con error" en `PancakePaginasComponent`: antes solo contaba `codigoError`, ignorando `errorSincronizacion` (por lo que páginas con token vencido u otro fallo de sincronización no se contabilizaban en ese KPI aunque sí mostraran el badge de error en la tabla) |

---

### Observaciones

- Es un componente puramente de presentación, igual que `PaginaDetalleModalComponent` de `gestion-paginas` (mismo patrón visual y clases CSS `upm-*`), pero de un solo modo — no necesita el `modo: 'usuarios' | 'paginas'` de aquel, porque este módulo solo lista páginas con error, nunca usuarios.
- El listado que recibe por `@Input() paginas` ya viene filtrado por el componente padre desde `paginasConError` (un `computed` que respeta los filtros/enfoque activos de la pantalla) — el modal en sí no sabe nada de `pageIdsActivos` ni de los demás filtros del dashboard.
