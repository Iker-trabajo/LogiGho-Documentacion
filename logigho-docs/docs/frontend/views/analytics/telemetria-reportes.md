---
autor: Iker Acevedo Vargas
fecha_creacion: 2026-09-21
ultima_actualizacion: 2026-10-28
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

## Cabecera

```text
Telemetría y auditoría de reportes          [Áreas] [Descargar] [Actualizar]
┌ Período  [Hoy|Ayer|Semana|Mes|Todo|Personalizado]   Tiempo en  [Automático ▾] ┐
```

Arriba van el título y las acciones; debajo, una barra de filtros con **Período** y **Tiempo en**. "Tiempo en" usa un desplegable propio (no el `<select>` nativo), con un ejemplo de cada unidad.

| Tarjeta | Qué muestra |
|---|---|
| Tiempo acumulado | Suma del tiempo de lectura de todos los usuarios en el período. Debajo, el usuario con mayor permanencia. |
| Tiempo medio de sesión | Promedio por sesión. Ignora las sesiones antiguas con tiempo `0` (anteriores a la corrección del tracker). |

---

## Unidad de tiempo

El selector **Tiempo en** cambia cómo se leen los tiempos en tarjetas y tablas:

| Opción | Ejemplo |
|---|---|
| Automático (por defecto) | Elige la unidad según el valor: `34 s`, `2 min`, `1h 5m`. |
| Segundos | `3.690 s` |
| Minutos | `61,5 min` |
| Horas | `1,03 h` |

La elección se guarda en el navegador (`localStorage`, clave `telemetria.unidadTiempo`) y **solo afecta la pantalla**: el Excel siempre se descarga en minutos.

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

**Descargar información** abre un modal para elegir **Excel**, **CSV** o **JSON**. Se exporta únicamente el resultado filtrado.

### Excel

La lógica vive en `services/utils/excel-telemetria.util.ts`. Según la pestaña activa se descarga uno de dos libros:

| Libro | Hojas | Columnas principales |
|---|---|---|
| `telemetria-reportes_<desde>_a_<hasta>.xlsx` | **Reportes**, **Información** | Reporte, Categoría, Tiempo total (min), Cantidad de clics. |
| `telemetria-usuarios_<desde>_a_<hasta>.xlsx` | **Usuarios**, **Consumo por reporte**, **Información** | Usuario, Correo, Fecha y hora de ingreso, Tiempo total (min), Clics, Consultas, Descargas, Última sesión. "Consumo por reporte" repite esos datos por cada usuario y reporte. |

Reglas comunes:

- Respetan el período, la búsqueda y la selección de gráficas.
- Máximo **1.000 filas** por hoja de resumen (`MAXIMO_FILAS_EXPORTACION`); el detalle por usuario y reporte llega hasta 5.000.
- Fechas y horas en hora de Bogotá; tiempos siempre en minutos.
- La hoja **Información** indica el período consultado y aclara que, antes de la corrección del tracker, el tiempo y los clics de las sesiones aparecen en `0`.

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
| Clics (`ConteoClics`) | Medir la interacción: clics en la página del visor y dentro del reporte. |

El tiempo se guarda en la misma sesión usando el `_id` de Mongo (ver [servicios](servicios.md#telemetriatrackerservice)). El último tramo, incluidas las visitas de menos de un minuto, se envía al salir del reporte aunque se cierre la pestaña.

No se registra el contenido interno del HTML ni las teclas del usuario; el seguimiento se limita a eventos de uso de la plataforma.

---

## Versión para celular

Los estilos están separados en `styles/telemetria-reportes.scss` (computador) y `styles/telemetria-reportes.mobile.scss` (celular), igual que en Dashboard Sin Despacho.

| Zona | En celular (≤768 px) |
|---|---|
| Página | Ocupa todo el ancho y no se desliza de lado (`overflow-x: clip`). |
| Altura | Sube 36 px para compensar el margen del contenedor (20 px) y el que deja el encabezado (`mb-4` menos el `-2%` del cuerpo); el título queda a 8 px de la barra. |
| Cabecera | Título con línea azul de acento, subtítulo corto y etiquetas en una línea. |
| Acciones | Fila de íconos: **Áreas**, **Descargar**, **Actualizar** (ícono arriba, texto corto abajo). |
| Filtros | Uno debajo del otro; los botones de período se deslizan de lado dentro de su recuadro. |
| Pestañas | Dos del mismo ancho, con el conteo debajo. |
| Indicadores | Solo los 2 principales, lado a lado; el resto con **"Ver N indicadores más"** (ver abajo). |
| Tablas | Cada fila es una tarjeta: nombre y estado arriba, tres cifras en cuadritos y una línea de detalle. |
| Detalle de usuario y descarga | Suben desde abajo como hoja inferior. |

Tamaños de pantalla usados: 1200, 900, 820, 768, 520 y 400 px.

### Indicadores en celular

Para no hacer scroll largo, al entrar solo se ven los dos indicadores más importantes de cada pestaña:

| Pestaña | Visibles al entrar | Al tocar "Ver más" |
|---|---|---|
| Resumen general | Visualizaciones, Usuarios activos | Tiempo medio, Descargas y Reporte líder (a lo ancho, al final). |
| Auditoría por usuario | Usuarios con acceso, Tiempo acumulado | Consultas activas, Descargas generadas. |

```text
[ Visualizaciones ] [ Usuarios activos ]
[ Tiempo medio    ] [ Descargas        ]
[ Reporte líder en tráfico (a lo ancho) ]
[            Ver menos ⌃               ]
```

- El botón **"Ver N indicadores más ⌄"** (borde punteado azul) cambia a **"Ver menos ⌃"**. El estado vive en `kpisAbiertos` y la clase `kpi-grid--abierto`.
- El orden y lo que se oculta se resuelven solo con CSS (`order` y `:nth-child`); el HTML de las tarjetas no cambia.
- En celular las tarjetas no muestran la minigráfica ni la nota inferior. En computador se ven todas, como antes.

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
| 2026-10-27 | Yadilton Torres | El tiempo de lectura ya se guarda; conteo de clics; tarjeta de tiempo acumulado; dos Excel (reportes y usuarios); selector "Tiempo en"; cabecera nueva con barra de filtros; versión para celular. |
| 2026-10-28 | Yadilton Torres | En celular, los 2 indicadores principales con "Ver más" y título más arriba. |
