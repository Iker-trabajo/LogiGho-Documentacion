# Módulo: Gestión de Páginas

---

## Autor: Adalberto González
Fecha creación: 2026-07-16  
Estado: desarrollo  
Tipo: módulo (1 vista + 1 componente hijo + 1 repositorio + 1 store)

---

## Índice

1. [Vista: GestionPaginasComponent](#1-vista-gestionpaginascomponent)
2. [Store: GestionPaginasStore](state/gestion-paginas-store.md)
3. [Servicio: GestionPaginasRepository](state/gestion-paginas-repository.md)
4. [Componente: UsuariosPaginaModalComponent](components/usuarios-pagina-modal.md)

---

## 1. Vista: GestionPaginasComponent

**Selector:** `app-gestion-paginas`  
**Ubicación:** `src/app/views/tienda/gestion-paginas/gestion-paginas.component.ts`  
**Acceso:** Autenticado | Roles con acceso total: `CEO`, `Desarrollador`

---

### ¿Qué hace? (para el usuario)

Es la pantalla de administración de las páginas de Pancake (canales de WhatsApp Business y Facebook) asociadas a las cuentas de Meta. Al abrirla, carga automáticamente todas las páginas registradas y muestra:

- **3 tarjetas KPI** en la parte superior: páginas registradas, páginas conectadas y usuarios activos (sumados en todas las páginas).
- **Una tabla paginada** con todas las páginas: nombre (con alerta si tiene `codigoError`), plataforma, estado de activación, estado de conexión, estado de suscripción, teléfono, país y cantidad de usuarios activos.
- **Filtros**: búsqueda de texto por nombre, 4 dropdowns de selección múltiple (Plataforma, Conexión, Suscripción, Páginas) con búsqueda interna, y un segmentado simple de activación (Todas / Activas / Inactivas).
- Al hacer clic en una fila se abre un modal con el detalle de usuarios asignados a esa página (activos y removidos, estos últimos diferenciados pero nunca ocultos).
- Un tour guiado explica cada sección de la pantalla paso a paso.

---

### Ruta

```
tienda/gestion-paginas
```

---

### Decoradores y configuración técnica

```typescript
@Component({
  selector: 'app-gestion-paginas',
  standalone: true,
  imports: [
    CommonModule, FormsModule,
    UsuariosPaginaModalComponent, TourGuiadoComponent
  ],
  templateUrl: './gestion-paginas.component.html',
  styleUrl: './gestion-paginas.component.scss',
})
export class GestionPaginasComponent implements OnInit, OnDestroy
```

---

### Control de acceso (RBAC)

Las acciones disponibles se resuelven en `cargarRol()` al inicializar, leyendo `sessionStorage['roles_asignados']` y cruzándolo con la constante `ROLES_GESTION_PAGINAS`:

| Rol | Acciones permitidas |
|---|---|
| `CEO`, `Desarrollador` | Todas (`*`) |
| Sin rol mapeado | Ninguna (solo lectura) |

Este módulo, a diferencia de `gestion-cuentas`, no tiene acciones de escritura sobre las filas (no hay crear/editar/activar páginas desde el frontend todavía) — `puede()` queda disponible para futuras acciones de escritura.

---

### Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `accionesPermitidas` | `string[] \| '*'` | Resultado de `cargarRol()`. `'*'` = acceso total |
| `modalAbierto` | `boolean` | Controla la visibilidad del modal de usuarios |
| `paginaEnModal` | `Pagina \| null` | Página cuyo detalle de usuarios se muestra en el modal |
| `showPlataformaDropdown` / `showConexionDropdown` / `showSuscripcionDropdown` / `showPaginaDropdown` | `boolean` | Visibilidad de cada dropdown de filtro (mutuamente excluyentes) |
| `searchPlataforma` / `searchConexion` / `searchSuscripcion` / `searchPagina` | `string` | Texto de búsqueda interna de cada dropdown; se limpia automáticamente al hacer clic fuera |
| `opcionesConexion` / `opcionesSuscripcion` | array fijo | Los 3 valores posibles de cada filtro (`Conectada/Desconectada/SinDato`, `Activa/Inactiva/SinDato`) |
| `tourAbierto` | `boolean` | Controla la visibilidad del tour guiado |
| `tourSteps` | `TourStep[]` | 4 pasos: KPIs, toolbar de filtros, tabla y botón de tour |

---

### Servicios y endpoints

| Servicio | Uso |
|---|---|
| `GestionPaginasStore` | Estado reactivo: listado, filtros, paginación, KPIs, opciones dinámicas de dropdown |
| `GestionPaginasRepository` | Acceso a datos (colección `PancakePaginas`) |

Ver detalle completo en [Store](state/gestion-paginas-store.md) y [Repositorio](state/gestion-paginas-repository.md).

---

### Filtros: dropdowns de selección múltiple

Los 4 dropdowns (Plataforma, Conexión, Suscripción, Páginas) siguen exactamente el mismo patrón visual y de interacción que `FiltrosInventarioComponent` de `resumen-inventario`:

- Cada opción es un checkbox; se pueden seleccionar varias a la vez (OR dentro del mismo filtro).
- Cada dropdown tiene su propio buscador interno que filtra las opciones visibles sin afectar la selección.
- Las opciones de **Plataforma** y **Páginas** son dinámicas: se calculan en el store (`plataformasDisponibles`, `nombresDisponibles`) a partir del listado real cargado, nunca están hardcodeadas.
- Un único `@HostListener('document:click')` (`onClickFuera`) cierra el dropdown correspondiente y limpia su buscador interno cuando el usuario hace clic fuera del `div` wrapper (identificado por `id`).
- Entre los distintos filtros multiselección (plataforma, conexión, suscripción, páginas) aplica AND; dentro de cada uno aplica OR.
- El filtro de activación (`activada`) es aparte, de un solo valor (`Activa | Inactiva | Todas`), como segmentado simple — no es multiselección porque solo tiene 2 estados reales + "todas".
- El botón "Limpiar" solo vacía los 4 filtros multiselección; no toca la búsqueda de texto ni el filtro de activación.

---

### Flujo principal

```
ngOnInit()
  ├─► cargarRol()     // Lee sessionStorage y resuelve accionesPermitidas
  └─► cargar()        // repo.getPaginas() → store.setPaginas()

Usuario escribe en el buscador
  └─► onBusqueda(valor) → store.setFiltros({ busqueda })

Usuario selecciona/deselecciona una opción en un dropdown
  └─► toggle<Filtro>(valor) → store.setFiltros({ <campo>: nuevoArray })

Usuario cambia el segmentado Activa/Inactiva/Todas
  └─► setActivada(valor) → store.setFiltros({ activada })

Usuario hace clic fuera de un dropdown abierto
  └─► onClickFuera(event) → cierra el dropdown correspondiente y limpia su búsqueda interna

Usuario hace clic en una fila de la tabla
  └─► abrirModalUsuarios(pagina)
        └─► modalAbierto = true, paginaEnModal = pagina
              └─► <app-usuarios-pagina-modal> muestra el detalle
```

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-16 | Adalberto González | Creación del módulo: listado de páginas de Pancake con KPIs, filtros dinámicos de selección múltiple, tour guiado y modal de detalle de usuarios |

---

### Observaciones

- El backend soporta pipelines de agregación Mongo reales vía el parámetro `pipeline` (aún en pruebas) y selección de campos vía `campos=` — este módulo usa `campos=` para excluir `tokenAccesoPagina` y `_id` desde el propio backend, nunca llegan al frontend.
