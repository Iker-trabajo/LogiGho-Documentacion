---
autor: Iker Acevedo Vargas
fecha_creacion: 2026-08-14
ultima_actualizacion: 2026-09-21
estado: desarrollo
nivel: 3
---

# Vista: Lista de Reportes

**Selector:** `app-lista-reportes`<br>
**Ubicación:** `src/app/views/analytics/lista-reportes`<br>
**Ruta:** `/app/analytics/vista-reportes`

---

## ¿Qué hace?

Es el catálogo que ve un usuario para encontrar y abrir reportes disponibles. Presenta tarjetas claras, adaptables a pantalla, con nombre completo, contexto, fecha y acceso rápido al visor.

---

## Cómo filtra el catálogo

1. Obtiene reportes desde `ReportesAnalytics`.
2. Conserva únicamente los que están `ACTIVO`.
3. Aplica los roles de la sesión: `Todos` significa acceso público dentro de la plataforma; de lo contrario debe existir al menos un rol común.
4. Aplica texto de búsqueda y área seleccionada.

Las chips de categoría se llenan desde las áreas organizacionales **activas**, no desde una lista hardcodeada ni valores encontrados accidentalmente en reportes antiguos.

| Elemento | Comportamiento |
|---|---|
| Buscador | Busca por nombre, descripción y área. |
| Filtro de área | Muestra `Todos` más áreas activas configuradas. |
| Tarjeta | Conserva nombres largos usando varias líneas; no los corta con puntos suspensivos. |
| Gestionar ETL | Solo se muestra a roles de gestión. |
| Ver reporte | Navega a `/app/analytics/vista-reportes/:ReporteId`. |

---

## Estados

| Estado | Mensaje al usuario |
|---|---|
| Cargando | Indicador de carga. |
| Sin reportes | “Todavía no hay reportes disponibles para ti”. |
| Sin coincidencias | Mensaje claro para ajustar búsqueda o filtro. |
| Error | Aviso de que no fue posible cargar el catálogo. |

---

## Importante sobre autorización

Este filtro mejora la experiencia, pero es un control de interfaz. El backend debe validar también `RolesPermitidos` al entregar el contenido del reporte; ese refuerzo queda como pendiente de seguridad de backend.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-08-14 | Iker Acevedo Vargas | Catálogo inicial, filtro por estado y rol. |
| 2026-09-21 | Iker Acevedo Vargas | Tarjetas responsive y categorías dinámicas desde Áreas Organizacionales. |
