# Módulo: Gestión de Cuentas

---

## Autor: Adalberto González
Fecha creación: 2026-07-15  
Estado: desarrollo  
Tipo: módulo (1 vista + 1 componente hijo + 1 repositorio + 1 store)

---

## Índice

1. [Vista: GestionCuentasComponent](#1-vista-gestioncuentascomponent)
2. [Store: GestionCuentasStore](state/gestion-cuentas-store.md)
3. [Servicio: GestionCuentasRepository](state/gestion-cuentas-repository.md)
4. [Componente: CuentaFormModalComponent](components/cuenta-form-modal.md)

---

## 1. Vista: GestionCuentasComponent

**Selector:** `app-gestion-cuentas`  
**Ubicación:** `src/app/views/tienda/gestion-cuentas/gestion-cuentas.component.ts`  
**Acceso:** Autenticado | Roles con acceso total: `CEO`, `Desarrollador`

---

### ¿Qué hace? (para el usuario)

Es la pantalla de administración de las cuentas de Pancake (perfiles de Meta) que la tienda usa para operar sus integraciones. Al abrirla, carga automáticamente todas las cuentas registradas y muestra:

- **3 tarjetas KPI** en la parte superior: cuentas registradas, cuentas activas y cuentas inactivas.
- **Una tabla paginada** con todas las cuentas: nombre, ID, token de acceso (truncado, con botón para copiar), estado, fecha de creación y fecha de última actualización.
- **Filtros** de búsqueda por texto (nombre o ID) y segmentado por estado (Todas / Activas / Inactivas).
- El operario puede crear una cuenta nueva (el ID se autogenera, nunca se digita a mano), editar el nombre y el token de una existente, y activar/desactivar cuentas con confirmación previa.
- Un tour guiado explica cada sección de la pantalla paso a paso.

---

### Ruta

```
tienda/gestion-cuentas
```

---

### Decoradores y configuración técnica

```typescript
@Component({
  selector: 'app-gestion-cuentas',
  standalone: true,
  imports: [
    CommonModule, ReactiveFormsModule,
    CuentaFormModalComponent, TourGuiadoComponent
  ],
  templateUrl: './gestion-cuentas.component.html',
  styleUrl: './gestion-cuentas.component.scss',
})
export class GestionCuentasComponent implements OnInit, OnDestroy
```

> No usa `ChangeDetectionStrategy.OnPush`: el `TourGuiadoComponent` mide posiciones de elementos con `getBoundingClientRect()`, y con OnPush el layout no siempre se repintaba a tiempo para ese cálculo, causando tirones visuales en el tour. Se usa detección por defecto, igual que en `GestionBodegasComponent`.

---

### Control de acceso (RBAC)

Las acciones disponibles se resuelven en `cargarRol()` al inicializar, leyendo `sessionStorage['roles_asignados']` y cruzándolo con la constante `ROLES_GESTION_CUENTAS`:

| Rol | Acciones permitidas |
|---|---|
| `CEO`, `Desarrollador` | Todas (`*`) |
| Sin rol mapeado | Ninguna (solo lectura) |

La columna **Acciones** de la tabla desaparece completamente si el usuario no tiene ninguna acción disponible sobre filas (`tieneAccionesTabla`).

---

### Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `accionesPermitidas` | `string[] \| '*'` | Resultado de `cargarRol()`. `'*'` = acceso total |
| `cuentaEditando` | `Cuenta \| null` | Cuenta en edición activa; `null` si es creación nueva |
| `showForm` | `Signal<boolean>` | Controla la visibilidad del modal de formulario |
| `formMode` | `Signal<FormMode>` | `'nueva'` o `'edicion'`, define el título y comportamiento del modal |
| `formGuardando` | `Signal<boolean>` | Deshabilita el botón Guardar mientras espera respuesta |
| `form` | `FormGroup` | Formulario reactivo con `nombre`, `tokenAcceso` y `estado` — **no incluye `cuentaId`**, ese campo nunca se muestra al usuario |
| `tourAbierto` | `boolean` | Controla la visibilidad del tour guiado |
| `tourSteps` | `TourStep[]` | 5 pasos: KPIs, toolbar, tabla, botón "Nueva cuenta" y botón de tour |

---

### Servicios y endpoints

| Servicio | Uso |
|---|---|
| `GestionCuentasStore` | Estado reactivo: listado, filtros, paginación, KPIs |
| `GestionCuentasRepository` | Acceso a datos (colección `PancakeCuentasPrincipales`) |

Ver detalle completo en [Store](state/gestion-cuentas-store.md) y [Repositorio](state/gestion-cuentas-repository.md).

---

### Flujo principal

```
ngOnInit()
  ├─► cargarRol()     // Lee sessionStorage y resuelve accionesPermitidas
  ├─► buildForm()     // Construye el FormGroup vacío
  └─► cargar()        // repo.getCuentas() → store.setCuentas()

Usuario hace clic en "Nueva cuenta"
  └─► abrirNueva()
        └─► buildForm() (sin cuenta) → showForm = true

Usuario hace clic en "Editar"
  └─► abrirEditar(cuenta)
        └─► buildForm(cuenta) → showForm = true

Usuario guarda el formulario
  └─► guardar()
        ├─► modo 'nueva'
        │     └─► repo.crearCuenta({ cuentaId: store.siguienteCuentaId(), ... })
        └─► modo 'edicion'
              ├─► repo.editarCuenta(id, dto)
              └─► store.upsertCuenta(...)   // optimistic update
        └─► cargar() // recarga completa para reflejar el estado real del backend

Usuario hace clic en "Desactivar/Activar"
  └─► onToggleEstado(cuenta)
        └─► SweetAlert2 pide confirmación
              └─► isConfirmed → repo.toggleEstado() + store.upsertCuenta()
```

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-15 | Adalberto González | Creación del módulo: CRUD de cuentas de Pancake con permisos dependiendo de el rol de el usuario, KPIs, tabla paginada, tour guiado y modal de formulario extraído a componente propio |
---

### Observaciones

- Las fechas (`fechaCreacion`, `fechaActualizacion`) se generan con `fechaColombiaISO()`, que resta 5 horas al `Date` actual antes de convertir a ISO (igual que `BodegaFormService.getCrearDto`), para que la hora mostrada en Mongo Compass coincida con la hora local de Colombia.
- El token de acceso se ingresa **manualmente** desde el formulario (se obtiene del panel de administración de Meta); no hay integración automática de sincronización ni de regeneración de token todavía.
