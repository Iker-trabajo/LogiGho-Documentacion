# Módulo: Gestión de Páginas

---

## Autor: Adalberto González
Fecha creación: 2026-07-16
Última actualización: 2026-07-24
Estado: desarrollo
Tipo: módulo (1 vista + 3 componentes hijos + 1 repositorio)

---

## Índice

1. [Vista: GestionPaginasComponent](#1-vista-gestionpaginascomponent)
2. [Servicio: GestionPaginasRepository](repository/gestion-paginas-repository.md)
3. [Componente: PaginaDetalleModalComponent](components/pagina-detalle-modal.md)
4. [Componente: ProductoSelectorModalComponent](components/producto-selector-modal.md)

---

## 1. Vista: GestionPaginasComponent

**Selector:** `app-gestion-paginas`
**Ubicación:** `src/app/views/tienda/gestion-paginas/gestion-paginas.component.ts`
**Acceso:** Autenticado | Roles con acceso total: `CEO`, `Desarrollador`

---

### ¿Qué hace?

Es la pantalla de administración de las páginas de Pancake (canales de WhatsApp Business y Facebook) asociadas a las cuentas de Meta. Al abrirla, carga automáticamente todas las páginas registradas y muestra:

- **3 tarjetas KPI** en la parte superior: páginas registradas, páginas conectadas y usuarios activos (sumados en todas las páginas).
- **Una tabla paginada** con todas las páginas: nombre (con alerta si tiene `codigoError`), plataforma, estado de activación, estado de conexión, estado de suscripción, teléfono, país, cantidad de usuarios activos y productos asociados.
- **Filtros**: búsqueda de texto por nombre, 4 dropdowns de selección múltiple (Plataforma, Conexión, Suscripción, Páginas) con búsqueda interna, y un segmentado simple de activación (Todas / Activas / Inactivas).
- Al hacer clic en una fila se abre un modal con el detalle de usuarios asignados a esa página.
- Al hacer clic en el botón **"Productos"** de una fila se abre un modal para elegir qué productos del catálogo despacha esa página.
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
    PaginaDetalleModalComponent, ProductoSelectorModalComponent,
    TourGuiadoComponent
  ],
  templateUrl: './gestion-paginas.component.html',
  styleUrl: './gestion-paginas.component.scss',
})
export class GestionPaginasComponent implements OnInit
```

> Este módulo ya no tiene un Store aparte. Antes existía `GestionPaginasStore`, pero se eliminó: la lista de páginas, los filtros, la paginación y los KPIs viven directamente en el componente como `signal()` y `computed()`.

---

### Control de acceso (RBAC)

Las acciones disponibles se resuelven en `cargarRol()` al inicializar, leyendo `sessionStorage['roles_asignados']` y cruzándolo con la constante `ROLES_GESTION_PAGINAS`:

| Rol | Acciones permitidas |
|---|---|
| `CEO`, `Desarrollador` | Todas (`*`) |
| Sin rol mapeado | Ninguna (solo lectura) |

Este módulo no tiene acciones de escritura sobre datos generales de la página (no hay crear/editar/activar páginas desde el frontend), pero sí permite **escribir** los productos asociados a cada página desde el modal de productos — esa acción no está bloqueada por `puede()` todavía, cualquier usuario autenticado que llegue a la pantalla puede usarla.

---

### Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `loading` / `error` | `Signal<boolean>` / `Signal<string \| null>` | Estado de carga y último error, expuestos como signals de solo lectura |
| `filtros` | `Signal<PaginasFiltros>` | Filtro de búsqueda + multiselección + activación |
| `pagina` | `Signal<number>` | Página actual de la tabla (paginación) |
| `plataformasDisponibles` / `nombresDisponibles` | `computed` | Opciones dinámicas de los dropdowns "Plataforma" y "Páginas", calculadas a partir de lo que ya se cargó |
| `paginasFiltradas` / `paginasPagina` / `totalPaginas` | `computed` | Resultado de aplicar todos los filtros, el slice de la página actual y el total de páginas de tabla |
| `kpis` | `computed` | `{ total, activadas, conectadas, usuariosActivos }` |
| `showPlataformaDropdown` / `showConexionDropdown` / `showSuscripcionDropdown` / `showPaginaDropdown` | `boolean` | Visibilidad de cada dropdown de filtro (mutuamente excluyentes) |
| `searchPlataforma` / `searchConexion` / `searchSuscripcion` / `searchPagina` | `string` | Texto de búsqueda interna de cada dropdown |
| `modalAbierto` / `modalModo` / `paginaEnModal` | — | Controlan el modal de detalle (usuarios de una página, o listado por KPI) |
| `productosDisponibles` | `ProductoAsociado[]` | Catálogo completo de productos (colección `Productos`), cargado una vez al iniciar la pantalla |
| `modalProductosAbierto` / `paginaEnModalProductos` / `guardandoProductos` | — | Controlan el modal de selección de productos asociados a una página |
| `tourAbierto` / `tourSteps` | — | Controlan el tour guiado |

---

### Servicios y endpoints

| Servicio | Uso |
|---|---|
| `GestionPaginasRepository` | Único punto de acceso a datos: lee y escribe la colección `PancakePaginas`, y lee el catálogo de la colección `Productos` |

Ver detalle completo en [Repositorio](repository/gestion-paginas-repository.md).

---

### Filtros: dropdowns de selección múltiple

Los 4 dropdowns (Plataforma, Conexión, Suscripción, Páginas) siguen el mismo patrón visual y de interacción que `FiltrosInventarioComponent` de `resumen-inventario`:

- Cada opción es un checkbox; se pueden seleccionar varias a la vez (OR dentro del mismo filtro).
- Cada dropdown tiene su propio buscador interno que filtra las opciones visibles sin afectar la selección.
- Las opciones de **Plataforma** y **Páginas** son dinámicas: se calculan con `computed()` a partir del listado real cargado, nunca están hardcodeadas.
- Un único `@HostListener('document:click')` (`onClickFuera`) cierra el dropdown correspondiente y limpia su buscador interno cuando el usuario hace clic fuera del `div` wrapper (identificado por `id`).
- Entre los distintos filtros multiselección aplica AND; dentro de cada uno aplica OR.
- El filtro de activación (`activada`) es aparte, de un solo valor (`Activa | Inactiva | Todas`), como segmentado simple.
- El botón "Limpiar" solo vacía los 4 filtros multiselección; no toca la búsqueda de texto ni el filtro de activación.

---

### Productos asociados a una página

Cada página puede tener asociados uno o varios productos del catálogo (colección `Productos`) — el/los producto(s) que esa página despacha.

- Cada fila de la tabla tiene un botón **"Productos"** (o "N productos" si ya tiene alguno asignado).
- Al hacer clic se abre `ProductoSelectorModalComponent`: un buscador desplegable con checkboxes (mismo patrón que el selector de productos de `gestion-bodegas`) y, debajo, la lista de productos ya asociados a esa página, cada uno con un botón para quitarlo.
- Al guardar, `GestionPaginasRepository.actualizarProductosAsociados()` reemplaza por completo el array `productosAsociados` del documento en `PancakePaginas` — no se suman productos nuevos a los viejos, se guarda la lista completa tal como quedó en el modal.
- Los productos asociados también se muestran en la vista `pancake-paginas`, dentro del cajón de detalle (`PaginaDetalleDrawerComponent`), en su pestaña "Productos".

---

### Flujo principal

```
ngOnInit()
  ├─► cargarRol()          // Lee sessionStorage y resuelve accionesPermitidas
  ├─► cargar()             // repo.getPaginas() → _paginas.set(...)
  └─► repo.getProductos()  // Carga el catálogo completo de productos (una sola vez)

Usuario escribe en el buscador
  └─► onBusqueda(valor) → setFiltros({ busqueda })

Usuario selecciona/deselecciona una opción en un dropdown
  └─► toggle<Filtro>(valor) → setFiltros({ <campo>: nuevoArray })

Usuario cambia el segmentado Activa/Inactiva/Todas
  └─► setActivada(valor) → setFiltros({ activada })

Usuario hace clic fuera de un dropdown abierto
  └─► onClickFuera(event) → cierra el dropdown correspondiente y limpia su búsqueda interna

Usuario hace clic en una fila de la tabla
  └─► abrirModalUsuarios(pagina)
        └─► modalAbierto = true, modalModo = 'usuarios', paginaEnModal = pagina
              └─► <app-pagina-detalle-modal> muestra el detalle de usuarios

Usuario hace clic en una tarjeta KPI
  └─► abrirKpiDetalle(tipo)
        └─► modalModo = 'paginas', modalKpiPaginas = [...] según el criterio del KPI

Usuario hace clic en "Productos" de una fila
  └─► abrirModalProductos(pagina)
        └─► modalProductosAbierto = true, paginaEnModalProductos = pagina
              └─► <app-producto-selector-modal> muestra el selector y la lista actual

Usuario guarda cambios en el selector de productos
  └─► guardarProductosAsociados(productos)
        └─► repo.actualizarProductosAsociados(pagina._id, productos)
              └─► Actualiza la lista local (_paginas) y cierra el modal
```

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-16 | Adalberto González | Creación del módulo: listado de páginas de Pancake con KPIs, filtros dinámicos de selección múltiple, tour guiado y modal de detalle de usuarios |
| 2026-07-24 | Adalberto González | Se elimina `GestionPaginasStore` (todo el estado pasa a vivir en el componente con signals); se reemplaza `UsuariosPaginaModalComponent` por `PaginaDetalleModalComponent` (dos modos: usuarios y listado por KPI); se agrega la posibilidad de asociar productos de la colección `Productos` a cada página mediante `ProductoSelectorModalComponent`, guardados en el campo `productosAsociados` de `PancakePaginas` |

---

### Observaciones

- El backend soporta pipelines de agregación Mongo reales vía el parámetro `pipeline` (aún en pruebas) y selección de campos vía `campos=` — este módulo usa `campos=` para excluir `tokenAccesoPagina` desde el propio backend.
- A diferencia de la versión anterior, el modelo `Pagina` **sí incluye `_id`** ahora (antes se excluía por seguridad). Fue necesario exponerlo porque `ConsumoGenericoService.actualizarGenerico()` siempre filtra la actualización por `_id` — no hay forma de decirle que filtre por `paginaId` en su lugar sin tocar ese servicio compartido por todo el proyecto.
