---
autor: Iker Acevedo Vargas
fecha_creacion: 2026-08-14
ultima_actualizacion: 2026-09-21
estado: desarrollo
nivel: 4
---

# Servicios: Analytics

**Ubicación:** `src/app/views/analytics/services`<br>
**Scope:** servicios reutilizados por las vistas del módulo.

---

## Mapa de responsabilidades

| Servicio o utilidad | Qué resuelve |
|---|---|
| `ReportesAnalyticsService` | Reportes, versiones, S3, compresión e integridad. |
| `ReporteSanitizerService` | Validación del HTML y CSP de seguridad. |
| `AreasOrganizacionalesService` | Áreas, normalización de nombres y asociación por roles. |
| `TelemetriaTrackerService` | Registro de una sesión real de lectura en el visor. |
| `TelemetriaConsultaService` | KPIs, gráficas, tabla y auditoría con agregaciones. |
| `formato.util.ts` | Fechas y tamaños en formato legible para español. |
| `roles.util.ts` | Lectura de roles de sesión y visibilidad de acciones de gestión. |

---

## `ReportesAnalyticsService`

Es la única puerta de datos para la gestión de reportes. Lista documentos, publica archivos comprimidos, calcula/valida SHA-256, obtiene versiones desde S3, actualiza metadatos y cambia estado.

| Método | Resultado visible |
|---|---|
| `listarReportes()` | Carga catálogo y gestión. |
| `publicarVersion()` | Crea reporte o agrega versión. |
| `actualizarMetadatos()` | Cambia información/roles sin crear versión. |
| `cambiarEstado()` | Archiva o reactiva. |
| `restaurarVersion()` | Define una versión anterior como vigente. |
| `obtenerHtmlVersion()` | Descarga, descomprime y verifica archivo antes de mostrarlo. |

---

## `AreasOrganizacionalesService`

Centraliza la colección `AreasOrganizacionales`. Normaliza nombres, evita duplicados en interfaz, conserva una caché corta y resuelve el área activa correspondiente a los roles del usuario. Para crear y editar usa operaciones genéricas sin auditoría, compatibles con una colección inicialmente vacía.

Ver modelo y reglas en [Áreas organizacionales](areas-organizacionales.md).

---

## Servicios de telemetría

### `TelemetriaTrackerService`

Inicia una sesión al mostrar un reporte y suma únicamente los segundos de pestaña visible. Envía un latido cada 60 segundos, considera activa una sesión reciente durante 150 segundos y sincroniza antes de eventos importantes o de abandonar la vista.

### `TelemetriaConsultaService`

Solicita KPIs, franjas horarias, distribución por área, reportes, usuarios y tabla paginada mediante agregaciones de base de datos. Filtra antes de agrupar y pagina en servidor para mantener una carga rápida.

---

## Compresión de respuestas de catálogos

El catálogo `Roles` se carga con el método genérico para evitar depender de un endpoint específico. El servicio detecta respuestas comprimidas en Zstandard y aplica una alternativa gzip cuando corresponde, por lo que la interfaz sigue consumiendo los  roles reales de la colección.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-08-14 | Iker Acevedo Vargas | Servicios iniciales de reporte, sanitización y utilidades. |
| 2026-09-21 | Iker Acevedo Vargas | Áreas organizacionales, seguimiento de consumo, consultas agregadas y carga robusta del catálogo de roles. |
