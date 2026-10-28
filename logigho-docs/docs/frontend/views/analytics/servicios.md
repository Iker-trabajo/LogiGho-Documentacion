---
autor: Iker Acevedo Vargas
fecha_creacion: 2026-08-14
ultima_actualizacion: 2026-10-28
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
| `excel-telemetria.util.ts` | Arma y descarga los dos Excel de telemetría (reportes y usuarios). |
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

**Cómo encuentra la sesión para actualizarla.** El endpoint de actualización espera el `_id` de Mongo, no el `SesionId` propio; con el `SesionId` respondía `500 ID no es válido` y el tiempo nunca se guardaba. Ahora:

```text
crear sesión (POST)
  → consultar con $match { SesionId } y leer su _id (descomprime si llega en gzip)
  → guardar ese _id en idMongo
latido cada 60 s
  → si todavía no hay idMongo, intenta resolverlo; si no, omite el envío
  → PUT con idMongo: segundos activos, último latido, descargas y clics
salir del reporte
  → último PUT con fetch + keepalive (actualizarGenericoSinAuditoriaAlSalir)
```

`keepalive` permite que el último envío llegue aunque se cierre la pestaña; así no se pierden las visitas de menos de un latido. El método vive en `ConsumoGenericoService` (`src/app/core/application/services/consumo-generico`).

**Clics.** `registrarClic()` suma uno a `ConteoClics`. Lo llaman el `@HostListener('click')` del visor y el aviso que envía el reporte desde el iframe (ver [Sandbox](reporte-sandbox.md#aviso-de-clics-dentro-del-reporte)). El conteo viaja con el latido siguiente.

### `TelemetriaConsultaService`

Solicita KPIs, franjas horarias, distribución por área, reportes, usuarios y tabla paginada mediante agregaciones de base de datos. Filtra antes de agrupar y pagina en servidor para mantener una carga rápida.

| Dato nuevo | Dónde aparece |
|---|---|
| Tiempo total acumulado y usuario de mayor permanencia | Tarjeta "Tiempo acumulado". |
| Tiempo total y clics por reporte | Tabla de reportes y Excel de reportes. |
| Clics y primer ingreso por usuario | Auditoría y Excel de usuarios. |
| `obtenerConsumoPorUsuarioReporte()` | Agrupa por usuario y reporte (hasta 5.000 filas) para la hoja "Consumo por reporte". |

- Los promedios ignoran las sesiones antiguas con tiempo `0`.
- Los nombres de usuario se toman siempre de la colección `Users`, para mostrar el nombre actual y no el que quedó guardado en la sesión.

---

## Pruebas

| Archivo | Qué comprueba |
|---|---|
| `telemetria-tracker.service.spec.ts` | Creación de la sesión, latidos con pestaña visible, guardado antes del primer latido, actualización con el `_id` de Mongo, clics acumulados sin un `PUT` por clic. |
| `consumo-generico.service.spec.ts` | `actualizarGenericoSinAuditoriaAlSalir`: `PUT` con `keepalive`, `_id` en la URL (con `?` o `&` según el método), token y cuerpo en JSON; si la red falla no lanza. |
| `telemetria-consulta.service.spec.ts` | Pipelines dentro del subconjunto de DocumentDB, paginación, filtros de gráfica, promedio que ignora sesiones con tiempo `0`, consumo agrupado por usuario y reporte, nombre vigente desde `Users` (y respaldo con el nombre de la sesión si `Users` no responde). |
| `excel-telemetria.util.spec.ts` | Filas y columnas de los dos Excel. |

---

## Compresión de respuestas de catálogos

El catálogo `Roles` se carga con el método genérico para evitar depender de un endpoint específico. El servicio detecta respuestas comprimidas en Zstandard y aplica una alternativa gzip cuando corresponde, por lo que la interfaz sigue consumiendo los  roles reales de la colección.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-08-14 | Iker Acevedo Vargas | Servicios iniciales de reporte, sanitización y utilidades. |
| 2026-09-21 | Iker Acevedo Vargas | Áreas organizacionales, seguimiento de consumo, consultas agregadas y carga robusta del catálogo de roles. |
| 2026-10-27 | Yadilton Torres | El tracker resuelve el `_id` de Mongo y guarda al salir con `keepalive`; conteo de clics; totales nuevos, consumo por usuario y reporte, nombres desde `Users` y utilidad de Excel. |
| 2026-10-28 | Yadilton Torres | Pruebas del guardado con `keepalive` y de las consultas nuevas de telemetría. |
