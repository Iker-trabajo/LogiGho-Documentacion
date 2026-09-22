---
autor: Iker Acevedo Vargas
fecha_creacion: 2026-09-21
ultima_actualizacion: 2026-09-21
estado: desarrollo
nivel: 4
---

# Vista: Telemetría y Auditoría de Reportes

**Selector:** `app-telemetria-reportes`<br>
**Ubicación:** `src/app/views/analytics/telemetria-reportes`<br>
**Ruta:** `/app/analytics/telemetria`<br>
**Acceso:** `telemetriaGuard` (`CEO`, `Jefe Datos` o `Desarrollador`).

---

## ¿Qué hace?

Permite entender qué reportes se usan, cuándo se consultan, qué áreas los consumen y cuánto tiempo permanecen activos los usuarios. Convierte el uso de reportes en información accionable sin descargar el historial completo al navegador.

| Pestaña | Para qué sirve |
|---|---|
| **Resumen general** | KPIs, reportes más consultados, horarios, distribución por área y tabla detallada. |
| **Auditoría por usuario** | Usuarios, consultas, tiempo acumulado, descargas, último acceso y actividad. |

---

## Filtros y comportamiento

| Control | Comportamiento |
|---|---|
| Hoy, Ayer, Semana, Mes | Consulta únicamente el período elegido. Recomendado para trabajo diario. |
| Todo | Consulta el histórico. Úselo de forma intencional porque puede crecer con el tiempo. |
| Rango personalizado | Permite elegir fechas libres. |
| Barras horarias | Un clic filtra por hora; `Ctrl`/`Cmd` + clic combina varias horas. |
| Dona de áreas | Un clic filtra por área; `Ctrl`/`Cmd` + clic combina áreas. |
| Limpiar | Quita selecciones visuales y vuelve al contexto actual. |

El mismo filtro actualiza tarjetas, gráficas y tabla. La interfaz comunica “Control + clic para seleccionar más de una sección”.

---

## Exportación

**Descargar información** abre un modal para elegir **CSV** o **JSON**. Se exporta únicamente el resultado filtrado.

---

## Qué se registra al consumir un reporte

Al abrir el visor se crea una sesión. Mientras la pestaña está visible se acumula tiempo real de lectura y se envían latidos periódicos. Al cambiar versión, descargar, actualizar, usar pantalla completa, ocultar la pestaña o salir, se sincroniza lo pendiente.

| Dato | Ejemplo de uso |
|---|---|
| Reporte, versión y fecha de inicio | Identificar qué se consultó y cuándo. |
| Usuario, roles y área | Entender el público consumidor. |
| Segundos activos y último latido | Medir permanencia sin contar tiempo oculto. |
| Descargas y acciones | Distinguir lectura, exportación e interacción. |
| Hora y fecha local | Construir franjas horarias. |

No se registra el contenido interno del HTML ni las teclas del usuario; el seguimiento se limita a eventos de uso de la plataforma.

---

## Rendimiento

La consulta se resuelve con agregaciones en la base de datos: filtra, agrupa, ordena y pagina antes de responder. El navegador recibe métricas y filas necesarias, no el histórico crudo completo.

- El período se consulta cuando cambia el rango o un filtro significativo; no al mover el mouse por una gráfica.
- Las tarjetas y gráficas reutilizan la selección activa.
- El tracker acumula fracciones visibles, evita contar pestañas ocultas y realiza cierre de mejor esfuerzo.

### Índices recomendados para producción

```javascript
use LogighoDB

db.SesionesReportesAnalytics.createIndexes([
  { key: { Inicio: -1 }, name: 'inicio_desc' },
  { key: { Area: 1, Inicio: -1 }, name: 'area_inicio' },
  { key: { HoraLocal: 1, Inicio: -1 }, name: 'hora_inicio' },
  { key: { Area: 1, HoraLocal: 1, Inicio: -1 }, name: 'area_hora_inicio' },
  { key: { UltimoLatido: -1 }, name: 'ultimo_latido_desc' }
])
```

| Índice | Acelera |
|---|---|
| `inicio_desc` | Filtros por período e histórico reciente. |
| `area_inicio` | Dona, filtros por área y tabla. |
| `hora_inicio` | Barras por franja horaria. |
| `area_hora_inicio` | Selecciones combinadas de área y hora. |
| `ultimo_latido_desc` | KPI de sesiones activas. |

---

## Colecciones relacionadas

| Colección | Responsabilidad |
|---|---|
| `SesionesReportesAnalytics` | Eventos y sesiones de consumo. |
| `ReportesAnalytics` | Metadatos, versiones y estado. |
| `AreasOrganizacionales` | Clasificación de áreas y roles. |
| `Roles` | Catálogo de roles. |
| `Users` | Nombre visible para auditoría cuando está disponible. |

---

## Verificación antes de producción

- Confirmar que los cinco índices fueron creados y no hay equivalentes duplicados.
- Abrir un reporte, esperar unos segundos, descargarlo y validar una sesión nueva en `SesionesReportesAnalytics`.
- Probar pestaña oculta y salida: el tiempo no debe crecer mientras no es visible.
- Validar Hoy, Semana, rango personalizado y multiselección en barras y dona.
- Probar la guardia con un usuario autorizado y otro no autorizado.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-09-21 | Iker Acevedo Vargas | Filtros inteligentes por período, gráficas multiselección, exportación CSV/JSON, seguimiento de consumo y consultas agregadas optimizadas. |
