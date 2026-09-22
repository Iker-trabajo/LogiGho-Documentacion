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

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-09-21 | Iker Acevedo Vargas | Se documentaron las rutas completas de Analytics, incluyendo telemetría y sus restricciones de acceso. |
