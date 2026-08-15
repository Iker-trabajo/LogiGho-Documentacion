---
autor: Iker Acevedo Vargas
fecha_creacion: 2026-08-14
ultima_actualizacion: 2026-08-14
estado: desarrollo
nivel: 3
---

# Vista: Lista de Reportes (Reportes ETL)

**Selector:** `app-lista-reportes`

**Ubicación:** `src/app/views/analytics/lista-reportes`

---

## ¿Qué hace?

Catálogo de reportes en tarjetas para cualquier usuario autenticado, filtrado por su rol de sesión. Es la puerta de entrada normal del usuario final al módulo (a diferencia de Gestión de Reportes, que es de administración).

---

## Ruta

| Ruta | Guard | Parámetros de URL |
| --- | --- | --- |
| `/app/analytics/vista-reportes` | `AuthGuard` | — |

---

## Propiedades clave

| Propiedad | Tipo | Descripción |
| --- | --- | --- |
| `reportes` | `ReporteAnalytics[]` | Ya filtrados por `Estado === 'ACTIVO'` y acceso de rol — nunca llegan aquí `BORRADOR`/`ARCHIVADO` |
| `categoriaSeleccionada` | `string` | Chip activo; `'Todos'` por defecto |
| `categoriasDisponibles` | `string[]` (getter) | Derivadas de las categorías presentes en `reportes` — no hardcodeadas |
| `puedeGestionar` | `boolean` (readonly) | `true` si el rol de sesión es `Jefe Datos` o `Desarrollador` — controla el botón "Gestionar ETL" |

---

## Servicios y endpoints

| Servicio | Método | Endpoint | Cuándo |
| --- | --- | --- | --- |
| `ReportesAnalyticsService` | `listarReportes()` | `GET metodoGenerico?coleccion=ReportesAnalytics` | Al inicializar |

Filtro de acceso resuelto en cliente (`roles.util.ts`), no en el backend — ver [limitación conocida](overview.md#limitación-conocida-fuera-del-alcance-del-módulo).

---

## Secciones de la vista

| # | Sección | Descripción |
| --- | --- | --- |
| 1 | **Header** | Título "Reportes ETL" + botón "Gestionar ETL" (solo roles autorizados) |
| 2 | **Buscador** | Filtra por nombre o categoría, sin debounce (dataset pequeño) |
| 3 | **Chips de categoría** | "Todos" + una por cada categoría presente en los reportes visibles |
| 4 | **Grid de tarjetas** | Responsive (`auto-fill, minmax(270px, 1fr)`) — badge de estado, categoría, título, descripción, fecha de actualización, versión, autor, botón "Ver Reporte" |

---

## Flujo principal

```
ngOnInit()
  -> cargarReportesVisibles()
     -> reportesService.listarReportes()
     -> obtenerRolesUsuario() (sessionStorage.roles_asignados)
     -> filtra: Estado === 'ACTIVO' && tieneAcceso(RolesPermitidos, rolesUsuario)
  -> reportes = resultado
```

**Regla de acceso** (`tieneAcceso`, idéntica al criterio que usa el sidebar en `default-layout.component.ts` para módulos):

```
si RolesPermitidos incluye "Todos" -> acceso
si no -> acceso solo si algún rol del usuario está en RolesPermitidos
```

---

## Estados de la vista

| Estado | Qué muestra |
| --- | --- |
| Cargando | Spinner |
| Con datos | Grid de tarjetas |
| Sin reportes visibles para el usuario | "Todavía no hay reportes disponibles para ti." |
| Búsqueda/filtro sin resultados | "Ningún reporte coincide con la búsqueda." |
| Error de carga | Alerta roja |

---

## Observaciones

- Un reporte en `BORRADOR` o `ARCHIVADO` **no aparece atenuado** — directamente no llega al arreglo `reportes`. No es "se ve pero deshabilitado" como en algunos mockups de referencia; es invisibilidad real.
- El botón "Ver Reporte" navega con `[routerLink]="['/app/analytics/vista-reportes', reporte.ReporteId]"` — con el prefijo `/app` explícito. Un `routerLink` sin ese prefijo (bug ya corregido) rompe la navegación porque `analytics` está anidado bajo `/app` en `app.routes.ts`.

---

## Changelog

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-08-14 | Iker Acevedo Vargas | Versión inicial: grid de tarjetas, filtro por categoría y rol, botón "Gestionar ETL" condicionado por rol |
