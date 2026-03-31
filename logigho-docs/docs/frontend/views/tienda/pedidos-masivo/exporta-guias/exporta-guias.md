---
autor: Iker
fecha_creacion: 2026-03-31
ultima_actualizacion: 2026-03-31
estado: desarrollo
nivel: 2
---

# Vista: Exportacion Masiva de Guias

**Autor:** Iker
**Selector:** `app-exporta-guias`
**Ubicación:** `SitioLogiGho/src/app/views/tienda/pedidos-masivo/exporta-guias`

---

## ¿Qué hace?

Permite a los usuarios consultar las guías generadas para sus tiendas y descargarlas en masa desde S3, agrupadas por transportadora. El usuario puede filtrar por carga completa, registro, IdCarga, estado, transportadora y tienda antes de exportar. Solo se descargan guías en estado `Cargado`.

---

## Ruta

| Propiedad | Valor |
|---|---|
| **Ruta** | `/tienda/exportar-guias-masivo` |
| **Título de página** | `Exporta-guias` |
| **Guard** | No definido en esta ruta (lazy load) |
| **Rol requerido** | Usuarios con tiendas asignadas (`tiendas_asignadas` en `sessionStorage`) |
| **Parámetros de URL** | Ninguno |

### Definición en `routes.ts`

```typescript
{
  path: 'exportar-guias-masivo',
  loadComponent: () =>
    import('./pedidos-masivo/exporta-guias/exporta-guias.component')
      .then(m => m.ExportaGuiasComponent),
  data: { title: 'Exporta-guias' }
}
```

---

## Estructura de archivos

```
exporta-guias/
├── exporta-guias.component.ts       Lógica principal
├── exporta-guias.component.html     Template de la vista
├── exporta-guias.component.scss     Estilos propios con variables SCSS
└── exporta-guias.component.spec.ts  Pruebas unitarias
```

---

## Secciones de la vista

| # | Sección | Condición de visibilidad | Descripción |
|---|---|---|---|
| 1 | **Encabezado** | Siempre | Header con gradiente azul, ícono de descarga y título |
| 2 | **Filtros** | Siempre | Card con selects de filtro. Antes de consultar, solo muestra el botón "Consultar Datos" |
| 3 | **Tabla de resultados** | `filteredRows.length > 0` | Tabla paginada con todas las guías filtradas y estadísticas de conteo por estado |
| 4 | **Sin resultados** | `filteredRows.length === 0 && dataLoaded` | Mensaje cuando los filtros no devuelven registros |
| 5 | **Estado inicial** | `!dataLoaded && !isLoading` | Mensaje orientativo para presionar "Consultar Datos" |

---

## Propiedades del componente

### Datos

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `rows` | `any[]` | `[]` | Todos los registros cargados desde la BD (sin `_id`) |
| `filteredRows` | `any[]` | `[]` | Subconjunto de `rows` con los filtros activos aplicados |
| `columns` | `{ key: string; title: string }[]` | `[]` | Columnas generadas dinámicamente desde las claves únicas de `rows` |

### Opciones de filtros (listas únicas)

| Propiedad | Tipo | Campo fuente |
|---|---|---|
| `uniqueCargasCompletas` | `string[]` | `IdCargaCompleta` |
| `uniqueRegistros` | `string[]` | `Registro` |
| `uniqueIdCargas` | `string[]` | `IdCarga` |
| `uniqueEstados` | `string[]` | `Estado` |
| `uniqueTransportadoras` | `string[]` | `TRANSPORTADORA` |
| `uniqueTiendas` | `string[]` | `Tienda` (ordenadas alfabéticamente) |

### Selección activa del usuario

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `selectedCargaCompleta` | `string` | `''` | Filtro por `IdCargaCompleta` |
| `selectedRegistro` | `string` | `''` | Filtro por `Registro` |
| `selectedIdCarga` | `string` | `''` | Filtro por `IdCarga` |
| `selectedEstado` | `string` | `'Cargado'` | Filtro por `Estado`. Inicia en `'Cargado'` por defecto |
| `selectedTransportadora` | `string` | `''` | Filtro por `TRANSPORTADORA` |
| `selectedTienda` | `string` | `''` | Filtro por `Tienda` |

### Estado UI

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `isLoading` | `boolean` | `false` | Activa el overlay de carga mientras se consultan datos |
| `isExporting` | `boolean` | `false` | Activa el overlay de exportación mientras se descargan guías |
| `dataLoaded` | `boolean` | `false` | Indica si ya se realizó al menos una consulta |

### Paginación

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `currentPage` | `number` | `1` | Página activa en la tabla |
| `pageSize` | `number` | `50` | Registros por página |
| `LOTE_MAX` | `number` | `4000` | Límite máximo de registros a traer por consulta |

### Constante de transportadoras

```typescript
readonly transportadoraOptions = [
  { value: 'INTERRAPIDISIMO', label: 'Interrapidísimo', bucket: 'bucket-guias-inter-prod'       },
  { value: 'ENVIA',           label: 'Envía',           bucket: 'bucket-guias-envia-prod'        },
  { value: 'D2E',             label: 'D2E',             bucket: 'bucket-guias-d2e-prod'          },
  { value: 'D2',              label: 'XCargo',          bucket: 'bucket-guias-xcargo-prod'       },
  { value: 'SERVIENTREGA',    label: 'Servientrega',    bucket: 'bucket-guias-servientrega-prod' }
]
```

Mapea el valor del campo `TRANSPORTADORA` de cada pedido al bucket S3 donde están almacenadas sus guías.

### Getters

| Getter | Tipo retorno | Descripción |
|---|---|---|
| `paginatedRows` | `any[]` | Filas de la página actual según `currentPage` y `pageSize` |
| `totalPages` | `number` | `Math.ceil(filteredRows.length / pageSize)` |

---

## Flujo de inicialización

```
ngOnInit()
  └─> (no hace nada, la carga es manual)

Usuario presiona "Consultar Datos"
  └─> consultarDatos()
        ├─> Verifica sessionStorage['tiendas_asignadas'] → si vacío, Swal y aborta
        ├─> isLoading = true
        ├─> fetchData()
        └─> isLoading = false, dataLoaded = true
```

La carga de datos **no ocurre automáticamente** al iniciar. El usuario debe presionar el botón "Consultar Datos" de forma explícita.

---

## Flujo de `fetchData()`

```
fetchData()
  └─> ConsumoGenericoService.consultarGenerico('metodoGenerico?coleccion=CargaPedido', {
        Tienda: sessionStorage['tiendas_asignadas'],
        mcomp: '2',
        lote: 4000
      })
        └─> DecompressionService.decompressZstd(response.Resultado)
              → rows = data sin campo _id
              → columns = todas las claves únicas de rows
              → uniqueCargasCompletas, uniqueEstados, uniqueRegistros, uniqueIdCargas,
                uniqueTransportadoras, uniqueTiendas (ordenadas)
              → filterRows()  ← aplica el filtro inicial Estado = 'Cargado'
```

---

## Flujo de `exportGuides()`

```
exportGuides()
  1. Valida que haya al menos un filtro de lote activo
     (selectedCargaCompleta, selectedRegistro o selectedIdCarga)
     → si no hay ninguno: Swal de advertencia y aborta

  2. Filtra filteredRows por Estado === 'Cargado'
     → si no hay ninguno: Swal de advertencia y aborta

  3. Agrupa NumeroPreenvio por TRANSPORTADORA
     → Map<transportadora, "pre1.txt,pre2.txt,...">

  4. isExporting = true

  5. Por cada transportadora con guías:
        └─> ConsumoGenericoService.insertarGenerico({
              bucketName: bucket de la transportadora,
              objectName: "pre1.txt,pre2.txt,...",
              boolCompress: false,
              boolBase64: false
            }, 'getMultiObject')
              └─> response.resultado = URL firmada de S3
                    → window.open(URL, '_blank')  ← descarga en nueva pestaña

  6. Todas las transportadoras se procesan en paralelo (Promise.all)

  7. isExporting = false
  8. Swal de éxito con cantidad de transportadoras descargadas
```

> Las transportadoras se procesan simultáneamente. Un error en una no interrumpe las demás; solo se registra en consola.

---

## Lógica de filtros en cascada

Los filtros tienen dependencias jerárquicas para evitar combinaciones inválidas:

```
CargaCompleta
  └─> resetea Registro, IdCarga, Tienda
      refresca uniqueRegistros, uniqueIdCargas, uniqueTiendas, uniqueTransportadoras

Tienda
  └─> resetea Registro, IdCarga
      refresca uniqueRegistros, uniqueIdCargas

Registro
  └─> resetea IdCarga
      refresca uniqueIdCargas

IdCarga / Estado / Transportadora
  └─> solo llaman a filterRows() directamente (no tienen hijos en la jerarquía)
```

Cuando el usuario selecciona una `CargaCompleta`, los selects de `Registro`, `IdCarga` y `Tienda` se reinician y sus opciones se reducen solo a los valores que existen dentro de esa carga. Esto evita combinaciones vacías.

---

## Métodos

### Públicos

| Método | Descripción |
|---|---|
| `ngOnInit()` | Ciclo de vida Angular. No realiza ninguna acción al iniciar |
| `consultarDatos()` | Valida tiendas asignadas, activa `isLoading` y llama a `fetchData()` |
| `fetchData()` | Consulta `CargaPedido`, descomprime la respuesta y puebla `rows`, `columns` y listas de filtros únicos |
| `onCargaCompletaChange()` | Maneja el cambio en el filtro CargaCompleta; refresca los filtros dependientes |
| `onTiendaChange()` | Maneja el cambio en el filtro Tienda; refresca Registro e IdCarga |
| `onRegistroChange()` | Maneja el cambio en el filtro Registro; refresca IdCarga |
| `filterRows()` | Aplica todos los filtros activos sobre `rows` y actualiza `filteredRows`. Resetea `currentPage` a 1 |
| `exportGuides()` | Agrupa guías por transportadora y descarga los archivos desde S3 en paralelo |
| `getTotalByEstado()` | Retorna un string con el conteo de registros filtrados por cada estado. Ej: `"Cargado: 120, Pendiente: 5"` |
| `nextPage()` | Avanza a la siguiente página si no es la última |
| `prevPage()` | Retrocede a la página anterior si no es la primera |

### Privados

| Método | Descripción |
|---|---|
| `extractUnique(rows, field)` | Extrae los valores únicos de un campo de un arreglo de filas, ordenados alfabéticamente |
| `refreshUniques(base)` | Recalcula `uniqueRegistros`, `uniqueIdCargas`, `uniqueTiendas` y `uniqueTransportadoras` a partir de un subconjunto de filas |

---

## Servicios utilizados

| Servicio | Métodos usados | Propósito |
|---|---|---|
| `ConsumoGenericoService` | `consultarGenerico()` | Consulta la colección `CargaPedido` con filtro por tiendas asignadas |
| `ConsumoGenericoService` | `insertarGenerico(..., 'getMultiObject')` | Solicita al backend las URLs firmadas de S3 para descargar múltiples guías por transportadora |
| `DecompressionService` | `decompressZstd()` | Descomprime la respuesta del endpoint `CargaPedido` |

---

## Endpoints que consume

| Método | Destino / Ruta | Parámetros | Cuándo |
|---|---|---|---|
| `GET` | `metodoGenerico?coleccion=CargaPedido` | `Tienda`, `mcomp: '2'`, `lote: 4000` | Al presionar "Consultar Datos" o "Refrescar Datos" |
| `POST` | `getMultiObject` | `bucketName`, `objectName` (lista de preenvíos), `boolCompress`, `boolBase64` | Al presionar "Exportar Guias", una llamada por transportadora con guías |

---

## Estados de la vista

| Estado | Condición | Qué muestra |
|---|---|---|
| **Inicial** | `!dataLoaded && !isLoading` | Tarjeta de estado vacío con instrucción de presionar "Consultar Datos" |
| **Cargando datos** | `isLoading = true` | Overlay full-screen con spinner y texto "Cargando datos..." |
| **Exportando** | `isExporting = true` | Overlay full-screen con spinner y texto "Descargando guías..." |
| **Con datos** | `filteredRows.length > 0` | Tabla paginada con estadísticas y controles de filtro completos |
| **Sin resultados** | `filteredRows.length === 0 && dataLoaded` | Mensaje de sin resultados con sugerencia de cambiar filtros |
| **Error de tiendas** | `tiendas_asignadas` vacío | Swal de advertencia, no carga datos |
| **Sin filtro de lote** | Exportar sin CargaCompleta/Registro/IdCarga | Swal de advertencia, no inicia la exportación |
| **Sin Cargados** | No hay filas con `Estado === 'Cargado'` | Swal de advertencia, no inicia la exportación |

---

## Validaciones antes de exportar

El botón "Exportar Guias" está **deshabilitado** si `isExporting` o `filteredRows.length === 0`. Adicionalmente, al ejecutar `exportGuides()` se valida:

1. Que al menos uno de `selectedCargaCompleta`, `selectedRegistro` o `selectedIdCarga` esté activo. Esto evita exportar accidentalmente la totalidad de guías sin un lote específico.
2. Que haya al menos un registro con `Estado === 'Cargado'` dentro de los filtros activos.

---

## Mecanismo de descarga de guías

Las guías no se descargan como archivos individuales. El flujo es:

1. Se construye una lista de `NumeroPreenvio` por transportadora separados por coma (ej. `"ABC123.txt,DEF456.txt"`).
2. Se llama al endpoint `getMultiObject` con esa lista y el bucket correspondiente.
3. El backend devuelve una **URL firmada de S3** que apunta al archivo ZIP o concatenado.
4. Se abre esa URL en una nueva pestaña del navegador (`window.open(..., '_blank')`).

Cada transportadora genera una descarga separada en una pestaña nueva.

---

## Estilos

Las variables SCSS son las mismas del módulo pedidos-masivo:

| Variable | Valor | Uso |
|---|---|---|
| `$primary-blue` | `#1e40af` | Color principal de botones, textos de sección |
| `$gradient-primary` | `linear-gradient(135deg, #1e40af → #1e3a8a)` | Header de la vista |
| `$success` | `#10b981` | Badge `status-ok` (estado Cargado / OK) |
| `$warning` | `#f59e0b` | Badge `status-pending` (estado Pendiente) |
| `$danger` | `#ef4444` | Badge `status-error` (estado ERROR) |

Los badges de estado se aplican en la tabla cuando el valor de la celda coincide exactamente con `'Cargado'`, `'OK'`, `'Pendiente'` o `'ERROR'`.

---

## Changelog de la vista

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-03-31 | Iker | Mejoras en rendimiento y módulo pedidos masivo |

---

## Observaciones

- **Carga manual obligatoria:** `ngOnInit` no carga datos. El usuario debe presionar "Consultar Datos" explícitamente. Esto evita consultas innecesarias al entrar a la vista.
- **Filtro de estado por defecto en `'Cargado'`:** Al cargar los datos, `filterRows()` se ejecuta con `selectedEstado = 'Cargado'`, por lo que la tabla siempre inicia mostrando solo guías listas para exportar.
- **Columnas dinámicas:** Las columnas de la tabla se generan recorriendo todas las claves de todos los registros (`Set` de claves). Si los registros tienen campos distintos entre sí, aparecerán todas las columnas posibles y las celdas sin valor mostrarán `-`.
- **Exportación solo para `Estado === 'Cargado'`:** Aunque el usuario pueda ver guías con otros estados en la tabla (cambiando el filtro de estado), la exportación siempre filtra internamente por `Estado === 'Cargado'` sobre `filteredRows`.
- **Límite de `LOTE_MAX = 4000`:** La consulta a `CargaPedido` trae máximo 4000 registros. Si una tienda tiene más de 4000 guías, los registros más antiguos pueden quedar fuera. No hay paginación server-side implementada.
- **Apertura de pestañas múltiples:** Al exportar varias transportadoras simultáneamente, el navegador abre varias pestañas a la vez. Algunos bloqueadores de pop-ups pueden impedir esto; el usuario debería permitir pop-ups del dominio.
- **Manejo de errores silencioso en exportación:** Si falla la descarga de una transportadora específica, el error solo se registra en consola. El usuario no recibe feedback de cuál falló, y el Swal final reporta éxito basado en cuántas tareas se intentaron, no en cuántas realmente descargaron.
