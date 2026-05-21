---

## Autor: Adalberto González

Fecha creacion: 2026-05-13
Estado: produccion
Tipo: vista

---

# Vista: ExportDataComponent + ExportRunnerComponent

Este módulo está compuesto por dos componentes que trabajan en tándem como una sola funcionalidad de exportación. `ExportDataComponent` es la pantalla de configuración donde el usuario arma la consulta, y `ExportRunnerComponent` es la pestaña que se abre automáticamente para ejecutar la descarga sin bloquear la UI principal.

---

## Componente — ExportRunnerComponent

**Selector:** `app-export-runner`  
**Ubicación:** `src/app/views/export-data/export-runner.component`  
**Acceso:** Público (sin `AuthGuard`) — el token JWT se recibe como query param `_token` y se inyecta al `sessionStorage` para que los servicios lo usen

---

### ¿Qué hace?

Se abre en una pestaña nueva al lanzar una exportación. Lee los parámetros de la URL, pagina la colección completa llamando repetidamente al backend (una página por vez), acumula los datos, genera el archivo Excel o PDF y lo descarga. Al finalizar cierra la pestaña automáticamente (2 segundos después de iniciar la descarga).

---

### Ruta

| Ruta | Guard | Parámetros de URL |
| --- | --- | --- |
| `/app/export-runner` | Sin guard (`isExportRunner` bypass en `AuthGuard`) | `coleccion`, `formato`, `exportId`, `orientacion`, `fechasFiltro`, `_token`, + filtros avanzados |

---

### Propiedades clave

| Propiedad | Tipo | Descripción |
| --- | --- | --- |
| `estado` | `'iniciando' \| 'procesando' \| 'finalizado' \| 'error'` | Estado visual que muestra el progreso al usuario |
| `mensaje` | `string` | Texto descriptivo del paso actual (ej: "Descargando página 3 de 10...") |

---

### Servicios y endpoints

| Servicio | Método | Endpoint | Cuándo |
| --- | --- | --- | --- |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET /metodoGenerico?coleccion=X&mcomp=2&lote=N&page=P&[filtros]&fechasFiltro=YYYYMMDD-YYYYMMDD` | Por cada página de datos hasta agotar `TotalRegistros` |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET /metodoGenerico?coleccion=ListaColecciones&NombreColeccion=X&Estado=ACTIVO` | Solo si `export-data` no envió `fechasFiltro` — para leer el `filtroFecha` configurado en la colección |
| `DecompressionService` | `decompressZstd()` / `decompressGzip()` | — | Para descomprimir cada respuesta paginada |

---

### Flujo principal

```
ngOnInit()
  -> lee queryParams: coleccion, formato, exportId, orientacion, fechasFiltro, filtros avanzados
  -> guarda _token en sessionStorage
  -> si params['servidor'] === 'true'
      -> procesarExportacionServidor(exportId)    // flujo alternativo vía localStorage
  -> flujo normal:
      -> si no hay fechasFiltro → GET ListaColecciones para leer filtroFecha configurado
      -> calcula lote desde localStorage[export_total_registros_<id>]
      -> GET página 1 → obtiene TotalRegistros real
      -> itera páginas 2..N acumulando en compressedDataArray[]
      -> descomprime y concatena todos los datos
      -> aplica filtro de columnas desde localStorage[export_columns_<id>]
      -> generateExcel() o generatePDF()
      -> saveAs() o doc.save()
      -> estado = 'finalizado'
      -> setTimeout(() => window.close(), 2000)
```

### Detalle del sistema de paginación

- El `lote` (tamaño de página) se toma de `localStorage['export_total_registros_<exportId>']` si existe
- La primera petición usa `page=1` y devuelve `TotalRegistros` real del backend
- Las páginas siguientes se calculan como `Math.ceil(TotalRegistros / lote)`
- Si el backend no devuelve `TotalRegistros` en la primera página, se asume que es la única página

### Preservación de tipos en Excel

Los valores `number` y `boolean` se escriben directamente en la celda sin convertirlos a string, para que Excel los trate como números y no como texto.

---

### Historial de cambios

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-05-13 | Adalberto González | Documentación inicial del módulo |
| 2026-05-13 | Adalberto González | Fix desfase UTC en filtros de fecha personalizados — se parsea YYYY-MM-DD como fecha local |
| 2026-05-13 | Adalberto González | Fix paginación: primera página usaba lote=88888 causando TotalRegistros=0 con filtros combinados |
| 2026-05-13 | Adalberto González | Fix filtro de tiendas asignadas: ahora aplica a todas las colecciones comparando por NombreTienda (no Id) |
| 2026-05-13 | Adalberto González | Auto-cierre de pestaña export-runner 2s después de iniciar descarga |
| 2026-05-13 | Adalberto González | Preservación de números y booleanos en Excel (no se convierten a string) |
| 2026-05-13 | Adalberto González | ConfiguracionCamposColeccion: campo Tienda corregido en 3 colecciones (MULTISELECT, CampoValorReferencia=NombreTienda) |

---

### Observaciones

- **`window.close()` solo funciona** porque `export-runner` es abierto via `window.open()` desde `export-data`.
- **`ConfiguracionCamposColeccion` es la fuente de verdad** para el comportamiento del módulo: tipo de control, opciones, referencia a colecciones externas y visibilidad de campos. Si un campo no aparece o se comporta mal, el primer lugar a revisar es esta colección en MongoDB.
- **El filtrado de tiendas es siempre en cliente** para los campos MULTISELECT que referencian `Tienda`. Esto significa que si la colección `Tienda` tiene miles de registros, se descargan todos y se filtran en memoria. Aceptable mientras el catálogo de tiendas sea pequeño (< 500).
- **Campos DATE desactivados** (`Estado: INACTIVO` en `ConfiguracionCamposColeccion`): los campos de tipo DATE de todas las colecciones están desactivados temporalmente porque el filtro de igualdad exacta no funciona con campos DATETIME del backend. El filtro de rango de fechas principal (`fechasFiltro`) sí funciona correctamente.
