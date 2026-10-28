---
autor: Iker Acevedo Vargas
fecha_creacion: 2026-08-14
ultima_actualizacion: 2026-10-28
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
| Cargando | Tarjetas fantasma animadas con la forma de las tarjetas reales. |
| Sin reportes | “Todavía no hay reportes disponibles para ti”. |
| Sin coincidencias | Mensaje claro para ajustar búsqueda o filtro. |
| Error | Aviso de que no fue posible cargar el catálogo. |

---

## Versión para celular

Estilos en `styles/lista-reportes.scss` (computador) y `styles/lista-reportes.mobile.scss` (celular).

| Zona | En celular (≤768 px) |
|---|---|
| Página | Ocupa todo el ancho. |
| Altura | Sube 36 px para compensar el margen del contenedor (20 px) y el que deja el encabezado (`mb-4` menos el `-2%` del cuerpo); el título queda a 8 px de la barra. |
| Título | "Reportes" con línea azul de acento; **?** y **Gestionar ETL** a la derecha, de 42 px. |
| Categorías | Chips que se deslizan de lado en una sola fila. |
| Tarjetas | Una columna, sin el salto del hover y con botón "Ver Reporte" táctil. |

Tamaños de pantalla usados: 860, 768 y 400 px.

---

## Importante sobre autorización

Este filtro mejora la experiencia, pero es un control de interfaz. El backend debe validar también `RolesPermitidos` al entregar el contenido del reporte; ese refuerzo queda como pendiente de seguridad de backend.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-08-14 | Iker Acevedo Vargas | Catálogo inicial, filtro por estado y rol. |
| 2026-09-21 | Iker Acevedo Vargas | Tarjetas responsive y categorías dinámicas desde Áreas Organizacionales. |
| 2026-10-27 | Yadilton Torres | Tarjetas fantasma de carga, estilos en `styles/` y versión para celular. |
| 2026-10-28 | Yadilton Torres | Título más arriba en celular. |
