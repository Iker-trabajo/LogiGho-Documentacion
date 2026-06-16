---

## Autor: Adalberto González

Fecha creacion: 2026-05-13
Estado: produccion
Tipo: vista

# Vista: ExportRunnerComponent

**Selector:** `app-export-runner`
**Ubicación:** `src/app/views/export-data/export-runner.component`
**Acceso:** `Administrador` `Tienda` `Controller` `CEO` `Dropshipper` `COO` `Logistica` `Accionista` `GLOBAL ACCOUNT` `TiendaNW` `BPO` `TRAFFICKER`

---

## ¿Qué hace?

Pestaña a parte encargada de generar y descargar exportaciones de datos de forma masiva. 
Recupera la información solicitada, aplica los filtros configurados por el usuario y genera archivos en distintos formatos para su descarga.

---

## Ruta

| Ruta | Guard | Parámetros de URL |
| --- | --- | --- |
| `/app/export-runner` | Sin guard (bypass `isExportRunner` en `AuthGuard`) | `coleccion`, `formato`, `exportId`, `orientacion`, `fechasFiltro`, `_token`, + filtros avanzados |

---

## Propiedades clave

| Propiedad | Tipo | Descripción |
| --- | --- | --- |
| `estado` | `'iniciando' \| 'procesando' \| 'finalizado' \| 'error'` | Estado visual que muestra el progreso al usuario |
| `mensaje` | `string` | Texto del paso actual, ej: "Descargando página 3 de 10..." |

---

## Servicios y endpoints

| Servicio | Método | Endpoint | Cuándo |
| --- | --- | --- | --- |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET /metodoGenerico?coleccion=X&mcomp=2&lote=N&page=P&[filtros]` | Por cada página hasta agotar `TotalRegistros` |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET /metodoGenerico?coleccion=ListaColecciones&NombreColeccion=X&Estado=ACTIVO` | Solo si `export-data` no envió `fechasFiltro` — lee `filtroFecha` configurado en la colección |
| `DecompressionService` | `decompressZstd()` / `decompressGzip()` | — | Para descomprimir cada respuesta paginada |

---

## Flujo principal

```
ngOnInit()
  -> lee queryParams: coleccion, formato, exportId, orientacion, fechasFiltro, filtros avanzados
  -> guarda _token en sessionStorage
  -> si params['servidor'] === 'true'
      -> procesarExportacionServidor(exportId)    // flujo alternativo vía body en localStorage
  -> flujo normal:
      -> si no hay fechasFiltro → GET ListaColecciones para leer filtroFecha configurado
      -> calcula lote desde localStorage[export_total_registros_<id>]
      -> GET página 1 → obtiene TotalRegistros real
      -> itera páginas 2..N acumulando compressedDataArray[]
      -> descomprime y concatena todos los datos
      -> aplica FILTROS_EXCLUSION por colección (ej: LiquidacionesLogigho excluye Estado="Impuesto Gobierno")
      -> aplica filtro de columnas desde localStorage[export_columns_<id>]
      -> generateExcel() | generateCSV() | generateJSON() | generatePDF()
      -> saveAs() o doc.save()
      -> estado = 'finalizado'
      -> setTimeout(() => window.close(), 2000)
```

---

## Historial de cambios

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-05-13 | Adalberto González | Creación del componente |
| 2026-06-16 | Adalberto González | Filtro de exclusion de los registros con estados "Impuesto Gobierno" para cuando se exporta por Liquidaciones |

---

## Observaciones

- **`window.close()` solo funciona** porque la pestaña fue abierta con `window.open()` desde `export-data`.
- **`FILTROS_EXCLUSION`** es el mecanismo para excluir filas por colección cuando el backend no soporta `$nin`. Se aplica después de descomprimir y antes del filtro de columnas. Para agregar exclusiones a otra colección, solo añadir una entrada al objeto.
- **El lote** se toma de `localStorage['export_total_registros_<exportId>']`. Si no existe, el backend decide el tamaño de página.
- **Preservación de tipos en Excel**: los valores `number` y `boolean` se escriben directamente en celda para que Excel los trate como números, no como texto.
