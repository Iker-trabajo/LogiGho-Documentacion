# Rutas y navegación

**Ubicación principal:** `src/app/app.routes.ts`<br>
**Rutas del módulo:** `src/app/views/analytics/routes.ts`

---

## Analytics — Reportes ETL

Todas las rutas siguientes viven bajo `/app/analytics` y requieren la sesión autenticada de la aplicación. La ruta base abre Gestión de Reportes para conservar el comportamiento actual del menú.

| Ruta | Vista | Acceso adicional | Para qué sirve |
|---|---|---|---|
| `/app/analytics` | Redirección | — | Envía a Gestión de Reportes. |
| `/app/analytics/gestion-reportes` | `GestionReportesComponent` | El acceso de publicación se controla desde la UI por rol. | Crear, versionar, editar y archivar reportes. |
| `/app/analytics/vista-reportes` | `ListaReportesComponent` | Reportes activos visibles para el usuario. | Catálogo de reportes. |
| `/app/analytics/vista-reportes/:id` | `VistaReporteComponent` | `id` es el `ReporteId`, no el `_id` de Mongo. | Abrir un reporte en el visor seguro. |
| `/app/analytics/telemetria` | `TelemetriaReportesComponent` | `telemetriaGuard`: `CEO`, `Jefe Datos` o `Desarrollador`. | Consultar consumo, actividad y auditoría. |

## Consideraciones de autorización

- El frontend oculta o muestra acciones según el rol de la sesión para que la experiencia sea clara.
- La autorización definitiva de publicar, editar roles o consultar reportes debe reforzarse en backend/API Gateway antes de abrir estos endpoints a consumidores no confiables. Es un pendiente conocido; no sustituye las validaciones de interfaz.

---

## AuthGuard y la ruta `/export-runner`

**Ubicación:** `src/app/core/application/services/guards/auth.guard.ts`

`AuthGuard` (`CanMatchFn`) protege las rutas de la aplicación: si no hay sesión válida, no deja entrar. La única excepción es **`/export-runner`** (exportación de datos), que entra sin validar sesión y toma el token que llega en la URL (`_token`) para guardarlo en `sessionStorage.id_token`.

| | Antes | Ahora (2026-10-09) |
|---|---|---|
| Cuándo se salta la validación | Si `export-runner` aparecía en **cualquier parte** de la URL: la ruta, el `pathname` o el hash (`#`) | Solo si la ruta es **exactamente** `/export-runner` (un único segmento) |
| Riesgo | Agregando `#export-runner` a otra ruta protegida (ej. `/app/tienda/wallet#export-runner`) se podía entrar sin sesión | Esa URL ahora manda al login |

```typescript
// Solo la ruta exacta /export-runner, según los segmentos de la URL que recibe el guard
const isExportRunner = segments.length === 1 && segments[0].path === 'export-runner';
```

> Pendiente (decisión del equipo): el token sigue llegando por la URL, donde puede quedar en el historial o en registros. Lo ideal sería un token de un solo uso o de corta duración.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-09-21 | Iker Acevedo Vargas | Se documentaron las rutas completas de Analytics, incluyendo telemetría y sus restricciones de acceso. |
| 2026-10-09 | Yadilton Torres | `AuthGuard`: solo la ruta exacta `/export-runner` entra sin sesión. Antes bastaba con que "export-runner" apareciera en cualquier parte de la URL, incluido el hash. |
