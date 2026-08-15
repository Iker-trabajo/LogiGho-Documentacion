---
autor: Iker Acevedo Vargas
fecha_creacion: 2026-08-14
ultima_actualizacion: 2026-08-14
estado: desarrollo
nivel: 4
---

# Vista: Visor de Reporte

**Selector:** `app-vista-reporte`

**Ubicación:** `src/app/views/analytics/vista-reporte`

---

## ¿Qué hace?

Renderiza un reporte HTML específico dentro del sandbox de seguridad, con cabecera de metadatos, controles de canvas (actualizar / pantalla completa), notas de la versión activa e historial de versiones (reciente + completo).

---

## Ruta

| Ruta | Guard | Parámetros de URL |
| --- | --- | --- |
| `/app/analytics/vista-reportes/:id` | `AuthGuard` | `id` = `ReporteId` (UUID propio, no el `_id` de Mongo) |

---

## Propiedades clave

| Propiedad | Tipo | Descripción |
| --- | --- | --- |
| `reporte` | `ReporteAnalytics \| null` | El reporte cargado por `id` de ruta |
| `versionSeleccionada` | `string \| null` | `VersionId` que se está mostrando (puede no ser la vigente) |
| `htmlEndurecido` | `string` | HTML con la CSP inyectada — lo único que recibe el sandbox |
| `htmlCrudoActual` *(privado)* | `string` | HTML original, sin CSP — se usa solo para "Descargar HTML" |
| `mostrarHistorialCompleto` | `boolean` | Controla el modal con todas las versiones (el panel lateral solo muestra las 3 últimas) |
| `puedeGestionar` | `boolean` (readonly) | Igual criterio que en Lista de Reportes — controla el botón "Gestionar ETL" |

---

## Servicios y endpoints

| Servicio | Método | Endpoint | Cuándo |
| --- | --- | --- | --- |
| `ReportesAnalyticsService` | `listarReportes()` | `GET metodoGenerico?coleccion=ReportesAnalytics` | Al inicializar (busca el reporte por `ReporteId`) |
| `ReportesAnalyticsService` | `obtenerHtmlVersion()` | `GET` S3 | Al cargar la versión activa o al cambiar de versión |
| `ReporteSanitizerService` | `endurecerHtml()` | — (cliente) | Sobre cada HTML crudo recibido, antes de pasarlo al sandbox |

---

## Secciones de la vista

| # | Sección | Descripción |
| --- | --- | --- |
| 1 | **Breadcrumb** | "Reportes › {nombre}" |
| 2 | **Header** | Título, metadatos (tamaño, fecha de generación, autor, badge "Activo"), botones "Gestionar ETL" (condicional) y "Descargar HTML" |
| 3 | **Canvas** | Header propio con "Actualizar" (re-fetch de la versión actual) y "Pantalla completa"; monta `ReporteSandboxComponent` con `[mostrarBotonPropio]="false"` |
| 4 | **Sidebar — Notas de Versión** | `Notas` de la versión activa, o mensaje de vacío |
| 5 | **Sidebar — Historial Reciente** | Últimas 3 versiones, clic cambia de versión sin recargar la página |
| 6 | **Modal — Historial completo** | Todas las versiones, mismo comportamiento de clic |

---

## Flujo principal

```
ngOnInit()
  -> id = ruta.snapshot.paramMap.get('id')
  -> cargarReporte(id)
     -> listarReportes() -> encuentra por ReporteId
     -> valida Estado === 'ACTIVO'
     -> cargarVersion(VersionActiva)
        -> obtenerHtmlVersion() -> verifica SHA-256 internamente
        -> htmlCrudoActual = html
        -> htmlEndurecido = sanitizer.endurecerHtml(html)
```

---

## Métodos clave

### `pantallaCompleta()`

Llama a `this.sandbox?.alternarPantallaCompleta()` vía `@ViewChild(ReporteSandboxComponent)` — el control real vive en el componente hijo; esta vista solo dispara el botón desde su propio header de canvas (con `mostrarBotonPropio=false` en el sandbox para no duplicar el botón flotante).

### `descargarHtml()`

Descarga el HTML **crudo** (`htmlCrudoActual`), nunca el endurecido — un archivo con `connect-src 'none'`/`default-src 'none'` inyectado se vería roto si alguien lo abriera suelto fuera de la plataforma.

---

## Estados de la vista

| Estado | Qué muestra |
| --- | --- |
| Cargando | Spinner "Cargando reporte…" |
| Reporte no existe / fue eliminado | Alerta roja |
| Reporte no está `ACTIVO` | Alerta "Este reporte no está disponible actualmente." |
| Cargando una versión distinta | Spinner "Renderizando…" dentro del canvas |
| Con datos | Header + canvas + sidebar completos |

---

## Observaciones

- Fechas y tamaños se formatean con funciones puras de `services/formato.util.ts` (`formatearFechaLarga`, `formatearTamano`) — sin registrar un locale global de Angular, que afectaría a toda la plataforma.
- No existe modo "pantalla completa" a nivel de página (`?full=1`) — se retiró junto con el botón de pestaña nueva; Fullscreen API lo reemplaza por completo.

---

## Changelog

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-08-14 | Iker Acevedo Vargas | Versión inicial: header con metadatos, canvas con fullscreen/refrescar, sidebar de notas e historial, descarga de HTML crudo |
