---
autor: Iker Acevedo Vargas
fecha_creacion: 2026-08-14
ultima_actualizacion: 2026-08-14
estado: desarrollo
nivel: 4
---

# Vista: Gestión de Reportes

**Selector:** `app-gestion-reportes`

**Ubicación:** `src/app/views/analytics/gestion-reportes`

---

## ¿Qué hace?

Panel de administración para el área de datos: subir un reporte HTML nuevo, publicar una versión nueva de uno existente, editar sus metadatos (incluyendo roles con acceso), ver su historial (por reporte y global), previsualizarlo dentro del mismo sandbox que ve el usuario final, y archivar/reactivar.

---

## Ruta

| Ruta | Guard | Parámetros de URL |
| --- | --- | --- |
| `/app/analytics/gestion-reportes` | `AuthGuard` | — |

Visibilidad del ítem de menú controlada por la colección `module` (rol requerido, no forzado por route guard — ver limitación conocida en [overview](overview.md)).

---

## Propiedades clave

| Propiedad | Tipo | Descripción |
| --- | --- | --- |
| `reportes` | `ReporteAnalytics[]` | Todos los reportes cargados (sin filtrar por rol — esta vista es de admin) |
| `reportesPagina` | `ReporteAnalytics[]` | Subconjunto de `reportesFiltrados` para la página actual |
| `terminoBusqueda` / `terminoBusquedaAplicado` | `string` | El input liga al primero; el filtrado real usa el segundo, actualizado con debounce de 200ms |
| `objetivoSubida` | `ReporteAnalytics \| null` | `null` = reporte nuevo; con valor = nueva versión o edición de ese reporte |
| `modoSoloMetadatos` | `boolean` | `true` cuando el modal es para editar metadatos sin subir archivo nuevo |
| `menuAbiertoId` / `menuPos` | `string \| null` / `{top, left}` | Estado del menú de acciones "⋮" por fila, posicionado con `getBoundingClientRect()` |
| `publicando` | `boolean` | Guardia de reentrancia — evita doble creación por doble clic |

---

## Servicios y endpoints

| Servicio | Método | Endpoint | Cuándo |
| --- | --- | --- | --- |
| `ReportesAnalyticsService` | `listarReportes()` | `GET metodoGenerico?coleccion=ReportesAnalytics` | Al inicializar y tras cada cambio |
| `ReportesAnalyticsService` | `publicarVersion()` | `PUT` S3 + `POST`/`PUT` Mongo | Al confirmar el modal (reporte nuevo o nueva versión) |
| `ReportesAnalyticsService` | `actualizarMetadatos()` | `PUT metodoGenerico?coleccion=ReportesAnalytics` | Al editar metadatos sin subir archivo |
| `ReportesAnalyticsService` | `cambiarEstado()` | `PUT metodoGenerico?coleccion=ReportesAnalytics` | Al archivar/reactivar |
| `ReportesAnalyticsService` | `restaurarVersion()` | `PUT metodoGenerico?coleccion=ReportesAnalytics` | Al hacer rollback desde el historial |
| `ReportesAnalyticsService` | `obtenerHtmlVersion()` | `GET` S3 | Al previsualizar |
| `ReporteSanitizerService` | `validar()` | — (cliente) | Al soltar/seleccionar un archivo |
| `ReporteSanitizerService` | `endurecerHtml()` | — (cliente) | Antes de montar el sandbox de previsualización |

---

## Secciones de la vista

| # | Sección | Descripción |
| --- | --- | --- |
| 1 | **Header** | Título, subtítulo, botones "Reglas para autores" (abre modal de reglas), "Historial" (historial global de actividad), "+ Nuevo Reporte" (dispara el mismo flujo de subida sin bajar hasta la zona de arrastre) |
| 2 | **Zona de arrastre** | Drag & drop + botón "Examinar archivos"; valida antes de abrir el modal de metadatos |
| 3 | **Tabla de reportes** | Nombre, categoría, última actualización, autor, estado (badge), roles visibles, menú de acciones "⋮" |
| 4 | **Paginador** | Selector de tamaño de página (5/10/25/50), navegación con elipsis — cabecera y footer en azul de marca |
| 5 | **Menú de acciones** | Ver reporte, editar metadatos, subir nueva versión, historial de versiones, archivar/reactivar |
| 6 | **Modal de metadatos** | Formulario de publicación/edición — ver [`ModalMetadatosReporteComponent`](#modalmetadatosreportecomponent) |
| 7 | **Modal de historial (por reporte)** | Lista de versiones de un reporte con opción de restaurar cualquiera anterior |
| 8 | **Modal de historial global** | Todas las versiones de todos los reportes, más recientes primero |
| 9 | **Modal de previsualización** | Monta `ReporteSandboxComponent` con el HTML endurecido de la versión seleccionada |

---

## Flujo principal

```
onFileInputChange() / onDrop()
  -> leerArchivoComoTexto()
  -> reporteSanitizer.validar(html, nombreArchivo)
     -> inválido: muestra hallazgos (línea + motivo), no continúa
     -> válido: abre modal de metadatos
  -> confirmarModal(datos)
     -> guardia: if (publicando) return
     -> modoSoloMetadatos ? actualizarMetadatos() : publicarVersion()
     -> cerrarModal() + cargarReportes()
     -> Swal.fire success
```

---

## Subcomponentes

| Componente | Selector | Qué hace |
| --- | --- | --- |
| `ModalMetadatosReporteComponent` | `app-modal-metadatos-reporte` | Formulario: nombre, descripción, categoría, estado, roles (consultados en vivo a la colección `Roles` de Mongo), notas de versión. Ver detalle abajo. |
| `ReglasAutoresReporteComponent` | `app-reglas-autores-reporte` | Modal flotante con la tabla de reglas de validación, generada desde `ReporteSanitizerService.obtenerReglasExpuestas()` — nunca se desincroniza del validador real |
| `ReporteSandboxComponent` | `app-reporte-sandbox` | Reutilizado aquí para la previsualización — ver [documentación del componente](reporte-sandbox.md) |

### `ModalMetadatosReporteComponent`

**Ubicación:** `gestion-reportes/components/modal-metadatos-reporte`

Los roles disponibles para el selector "¿Quién puede verlo?" se consultan en `ngOnInit()` contra `RolesService.consultarRoles()` (colección `Roles` real de Mongo, la misma que usa `administracion/roles`) — nunca están hardcodeados, así que un rol nuevo creado en esa colección aparece automáticamente la próxima vez que se abre el modal. Si la consulta falla, el formulario sigue usable con al menos la opción `Todos`.

El botón de publicar se deshabilita mientras `[publicando]="true"` (input recibido del padre) — corrige un bug de doble clic que creaba el reporte duplicado.

---

## Estados de la vista

| Estado | Qué muestra |
| --- | --- |
| Cargando | Spinner + texto "Cargando reportes…" |
| Con datos | Tabla paginada |
| Vacío (sin reportes) | "Todavía no hay reportes publicados." |
| Vacío (búsqueda sin match) | "Ningún reporte coincide con la búsqueda." |
| Error de validación de archivo | Alerta roja con lista de hallazgos (línea + motivo + extracto) |
| Error de guardado | Alerta roja genérica |

---

## Observaciones

- **Escape cierra todo:** un único `@HostListener('document:keydown.escape')` cierra, en orden, el elemento más reciente abierto (menú de acciones → preview → historial global → historial por reporte → modal de metadatos → reglas).
- **Sin "abrir en pestaña nueva":** decisión deliberada, ver [overview](overview.md#decisión-descartada-abrir-en-pestaña-nueva).
- **Nunca se borra nada de S3:** archivar un reporte solo cambia `Estado` en Mongo; las keys de versiones anteriores permanecen indefinidamente.

---

## Changelog

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-08-14 | Iker Acevedo Vargas | Versión inicial: subida, edición, historial (por reporte y global), paginación, roles dinámicos desde Mongo, guardia de doble envío |
