# Módulo: Gestión de Bodegas

---

## Autor: Adalberto González
Fecha creación: 2026-06-24  
Estado: desarrollo  
Tipo: módulo (1 vista + 5 componentes hijos + 1 orquestador + 1 repositorio + 1 store)

---

## Índice

1. [Vista: GestionBodegasComponent](gestion-bodegas.md)
2. [Servicio: BodegasOrchestratorService](services/bodegas-orchestrator.md)
3. [Servicio: BodegasRepository](services/bodegas-repository.md)
4. [Store: BodegasStore](services/bodegas-store.md)
5. [Componente: BodegaCardComponent](components/bodega-card.md)
6. [Componente: BodegaFormModalComponent](components/bodega-form-modal.md)
7. [Componente: AsignacionModalComponent](components/asignacion-modal.md)
8. [Componente: AsignacionesDrawerComponent](components/asignaciones-drawer.md)
9. [Componente: KpiBodegasModalComponent](components/kpi-bodegas-modal.md)
10. [ADR-001 — Patrón de arquitectura](adr/ADR-001-arquitectura.md)

---

## Vista: GestionBodegasComponent

**Selector:** `app-gestion-bodegas`  
**Ubicación:** `src/app/views/logistica/gestion-bodegas/gestion-bodegas.component.ts`  
**Acceso:** Autenticado | Roles: `Administrador`, `COO`, `CEO`, `Desarrollador`, `Jefe Logistica`, `Logistica`, `Bodega`

---

### ¿Qué hace? (para el usuario)

Es la pantalla principal del módulo de bodegas. Al abrirla, carga automáticamente todos los datos y muestra:

- **3 tarjetas KPI** en la parte superior: bodegas registradas, stock total en bodegas y bodegas con bajo stock. Cada tarjeta es clicable y abre un modal con el listado detallado de bodegas correspondientes.
- **Una tabla paginada** con todas las bodegas, su stock actual, estado y botones de acción por fila.
- **Filtros** de búsqueda por texto y segmentado por estado (Todas / Activas / Inactivas).
- El operario puede crear nuevas bodegas, editarlas, habilitarlas o deshabilitarlas (con confirmación), asignar productos y gestionar las asignaciones existentes de cada bodega.
- Las acciones disponibles dependen del rol del usuario: algunos solo pueden ver, otros pueden gestionar asignaciones, y los administradores tienen acceso completo.

---

### Ruta

```
logistica/gestion-bodegas
```

---

### Decoradores y configuración técnica

```typescript
@Component({
  selector: 'app-gestion-bodegas',
  standalone: true,
  imports: [
    CommonModule, ReactiveFormsModule,
    BodegaCardComponent, BodegaFormModalComponent,
    AsignacionModalComponent, AsignacionesDrawerComponent,
    KpiCardComponent, KpiBodegasModalComponent
  ],
  templateUrl: './gestion-bodegas.component.html',
  styleUrl: './gestion-bodegas.component.scss',
})
export class GestionBodegasComponent implements OnInit
```

---

### Control de acceso (RBAC)

Las acciones disponibles se resuelven en `cargarRol()` al inicializar, leyendo `sessionStorage['roles_asignados']` y cruzándolo con la constante `ROLES_BODEGAS`:

| Rol | Acciones permitidas |
|---|---|
| `Administrador`, `CEO`, `Desarrollador`, `Lider de Logistica` | Todas (`*`) |
| `Logistica` | `asignar`, `gestionAsignaciones`, `editarAsignacion`, `eliminarAsignacion` |
| `Bodega` | `gestionAsignaciones` |
| Sin rol mapeado | Ninguna (solo lectura) |

La columna **Acciones** de la tabla desaparece completamente si el usuario no tiene ninguna acción disponible sobre filas.

---

### Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `accionesPermitidas` | `string[] \| '*'` | Resultado de `cargarRol()`. `'*'` = acceso total |
| `bodegaEnEdicion` | `Bodega \| null` | Bodega en edición activa; `null` si es creación nueva |
| `drawerAbierto` | `boolean` | Controla la visibilidad del modal de gestión de asignaciones |
| `bodegaDrawer` | `Bodega \| null` | Bodega cuyos productos se están gestionando |
| `nombresProducto` | `Map<string, string>` | Mapa `idProducto → nombre` cargado al abrir el modal de asignaciones |
| `kpiDetalleAbierto` | `boolean` | Controla la visibilidad del modal de detalle KPI |
| `kpiDetalleBodegas` | `Bodega[]` | Lista de bodegas ya filtrada que se pasa al modal KPI |

---

### Flujo principal

```
ngOnInit()
  ├─► cargarRol()               // Lee sessionStorage y resuelve accionesPermitidas
  └─► orchestrator.iniciar()    // Carga paralela de 4 fuentes
        ├─► getBodegas()              → store.setBodegas() (ordenadas desc por fechaCreacion)
        ├─► getCiudades()             → orchestrator.ciudades
        ├─► getTodasLasAsignaciones() → base para calcular stock
        └─► getStockPorProducto()     → ResumenInventario sin filtro de tienda
              └─► calcularStockPorBodega() → Map<idBodega, stockTotal>

Usuario hace clic en una tarjeta KPI
  └─► abrirKpiDetalle(tipo)
        └─► filtra store.bodegas() según tipo → kpiDetalleBodegas
            kpiDetalleAbierto = true → modal visible

Usuario hace clic en "Deshabilitar"
  └─► onToggleEstado(bodega)
        └─► SweetAlert2 pide confirmación
              └─► isConfirmed → orchestrator.toggleEstado(bodega)

Usuario abre gestión de asignaciones
  └─► abrirDrawer(bodega)
        ├─► orchestrator.cargarProductos()         // caché, solo 1 vez
        ├─► nombresProducto = Map(productos)
        └─► orchestrator.cargarAsignacionesBodega(bodega.idBodega)
```

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-06-24 | Adalberto González | Creación del módulo completo con RBAC, KPIs, tabla paginada y gestión de asignaciones |
| 2026-06-25 | Adalberto González | Agregado stock individual por producto en el drawer de asignaciones |
| 2026-06-25 | Adalberto González | Guardado de `idTienda` y `nombreTienda` en asignaciones; navegación a resumen-inventario desde modal KPI |

---

### Observaciones

- `puede(accion)` evalúa `accionesPermitidas` en cada llamada. Se usa en el template con `*ngIf="puede('crear')"` etc.
- La confirmación de deshabilitar/habilitar usa SweetAlert2 con color de botón dinámico (rojo para deshabilitar, azul para habilitar).
- Las bodegas se ordenan descendente por `fechaCreacion` (más recientes primero) dentro del store.
