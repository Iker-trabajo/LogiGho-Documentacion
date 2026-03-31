---
autor: Iker
fecha_creacion: 2026-03-31
ultima_actualizacion: 2026-03-31
estado: desarrollo
nivel: 2
---

# Vista: Generación Masiva de Guías

**Autor:** Iker
**Selector:** `app-genera-guias`
**Ubicación:** `SitioLogiGho/src/app/views/tienda/pedidos-masivo/genera-guias`

---

## ¿Qué hace?

Permite a los usuarios consultar los pedidos cargados en el sistema (colección `CargaPedido`), filtrarlos por múltiples criterios y disparar la generación masiva de guías contra la API de la transportadora. También permite lanzar validaciones previas de pedidos y exportar los registros filtrados a CSV.

---

## Ruta

| Propiedad | Valor |
|---|---|
| **Ruta** | `/tienda/genera-guias-masivo` |
| **Título de página** | `Genera-guias` |
| **Guard** | No definido en esta ruta (lazy load) |
| **Rol requerido** | Usuarios con tiendas asignadas (`tiendas_asignadas` en `sessionStorage`) |
| **Parámetros de URL** | Ninguno |

### Definición en `routes.ts`

```typescript
{
  path: 'genera-guias-masivo',
  loadComponent: () =>
    import('./pedidos-masivo/genera-guias/genera-guias.component')
      .then(m => m.GeneraGuiasComponent),
  data: { title: 'Genera-guias' }
}
```

---

## Estructura de archivos

```
genera-guias/
├── genera-guias.component.ts       Lógica principal
├── genera-guias.component.html     Template de la vista
├── genera-guias.component.scss     Estilos propios
└── genera-guias.component.spec.ts  Pruebas unitarias
```

---

## Secciones de la vista

| # | Sección | Descripción |
|---|---|---|
| 1 | **Panel de filtros** | Selectores encadenados: Carga Completa, Registro, IdCarga, Estado, Tienda + selector de tamaño de etiqueta |
| 2 | **Barra de acciones** | Botones: Consultar datos, Generar Guías, Validar Pedidos, Exportar CSV. Toggle de alarmas |
| 3 | **Resumen de estados** | Texto dinámico con conteo de registros agrupados por estado (e.g. `Pendiente: 12, Cargado: 8`) |
| 4 | **Tabla de pedidos** | Tabla paginada con columnas fijas que muestra los registros filtrados de `CargaPedido` |
| 5 | **Paginación** | Botones Anterior / Siguiente con indicador de página actual sobre total |

---

## Propiedades del componente

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `rows` | `CargaPedido[]` | `[]` | Todos los registros traídos del backend |
| `filteredRows` | `CargaPedido[]` | `[]` | Subconjunto de `rows` tras aplicar los filtros activos |
| `columns` | `{ key: string; title: string }[]` | `[]` | Columnas fijas para renderizar la tabla |
| `tamanos` | `TipoEtiqueta[]` | `[]` | Tipos de etiqueta disponibles cargados al inicio |
| `uniqueCargasCompletas` | `string[]` | `[]` | Valores únicos de `IdCargaCompleta` (siempre globales, no filtrados) |
| `uniqueIdCargas` | `string[]` | `[]` | Valores únicos de `IdCarga` según filtros superiores activos |
| `uniqueRegistros` | `string[]` | `[]` | Valores únicos de `Registro` según filtros superiores activos |
| `uniqueEstados` | `string[]` | `[]` | Valores únicos de `Estado` (siempre globales) |
| `uniqueTiendas` | `string[]` | `[]` | Valores únicos de `Tienda` según filtros superiores activos (ordenados) |
| `selectedCargaCompleta` | `string` | `''` | Filtro activo de Carga Completa |
| `selectedIdCarga` | `string` | `''` | Filtro activo de IdCarga |
| `selectedRegistro` | `string` | `''` | Filtro activo de Registro |
| `selectedEstado` | `string` | `''` | Filtro activo de Estado |
| `selectedSize` | `string` | `''` | Id del tipo de etiqueta seleccionado |
| `selectedTienda` | `string` | `''` | Filtro activo de Tienda |
| `toggleChecked` | `boolean` | `true` | Si es `true` se generan guías con alarmas activadas |
| `isLoading` | `boolean` | `false` | Deshabilita controles y activa feedback visual de carga |
| `dataLoaded` | `boolean` | `false` | Indica si se han traído datos al menos una vez |
| `currentPage` | `number` | `1` | Página activa en la tabla |
| `pageSize` | `number` | `50` | Registros por página |

### Constantes de módulo

| Constante | Valor | Descripción |
|---|---|---|
| `ENDPOINT_CARGA_PEDIDO` | `'metodoGenerico?coleccion=CargaPedido'` | Endpoint de consulta de pedidos |
| `ENDPOINT_TIPO_ETIQUETA` | `'metodoGenerico?coleccion=TipoEtiqueta'` | Endpoint de tipos de etiqueta |
| `ENDPOINT_CARGA_PREENVIOS` | `'cargaPreenvios'` | Endpoint de generación de guías |
| `ENDPOINT_VALIDACION` | `'validacionPedidos'` | Endpoint de validación de pedidos |
| `LOTE_MAX` | `4000` | Límite de registros por consulta al backend |

### Getters

| Getter | Tipo retorno | Descripción |
|---|---|---|
| `canGenerateGuias` | `boolean` | `true` si: `selectedSize` definido, no `isLoading`, `selectedCargaCompleta` o `selectedIdCarga` activo, y al menos 2 filtros activos |
| `generarGuiasTitle` | `string` | Tooltip descriptivo del botón Generar Guías según el estado de los filtros |
| `paginatedRows` | `CargaPedido[]` | Filas de la página actual |
| `totalPages` | `number` | `Math.ceil(filteredRows.length / pageSize)` |

---

## Flujo de inicialización

```
ngOnInit()
  └─> fetchTamanios()
        └─> ConsumoGenericoService.consultarGenerico('1', ENDPOINT_TIPO_ETIQUETA)
              └─> decompressionService.decompressGzip(response.Resultado)
              → Pobla this.tamanos[] con { Nombre, Id }
```

---

## Flujo principal: `consultarDatos()`

Se ejecuta al hacer clic en "Consultar datos".

```
consultarDatos()
  1. Verifica sessionStorage['tiendas_asignadas'] → si vacío, Swal de advertencia y aborta
  2. isLoading = true, limpia idsEnProceso
  3. fetchData()
        └─> ConsumoGenericoService.consultarGenerico('1', ENDPOINT_CARGA_PEDIDO, {
              Tienda: sessionStorage['tiendas_asignadas'], mcomp: '2', lote: 4000
            })
              └─> decompressionService.decompressZstd(response.Resultado)
              → Elimina campo _id de MongoDB de cada fila
              → Pobla this.rows, this.filteredRows
              → Genera columnas desde COLUMNAS_FIJAS (columnas estáticas predefinidas)
              → Calcula uniqueCargasCompletas, uniqueEstados (globales)
              → Calcula uniqueRegistros, uniqueIdCargas, uniqueTiendas via refreshUniques()
              → Aplica filterRows()
  4. dataLoaded = true
  5. isLoading = false
```

---

## Flujo principal: `generateGuias()`

Se ejecuta al hacer clic en "Generar Guías".

```
generateGuias()
  1. validarSeleccion('generar guías') → requiere CargaCompleta o IdCarga + mín. 2 filtros
  2. Si toggleChecked === false → Swal de confirmación
  3. executeGuideGeneration()
        1. Filtra rowsAProcesar: Estado='Pendiente' o (Estado='Cargado' sin NumeroPreenvio)
        2. Si rowsAProcesar.length === 0 → Swal 'Sin registros procesables' y aborta
        3. Extrae idsCargas únicos de rowsAProcesar
        4. Bloquea si algún idCarga está en idsEnProceso → Swal 'Proceso en curso'
        5. Si pendientes > 50 → Swal de advertencia de larga duración con estimación de tiempo
           (minutosEstimados = Math.ceil(pendientes / 60))
        6. Si pendientes ≤ 50 pero hay excluidas → Swal informativo de exclusiones
        7. isLoading = true
        8. Para cada idCarga: POST cargaPreenvios { IdCarga, TipoEtiqueta, CargarAlarmas }
           → Concurrencia máxima: 2
           → 504 → estado 'EN_PROCESO', agrega a idsEnProceso
           → Otro error → estado 'ERROR'
           → OK → estado 'OK'
        9. mostrarResumen('Generación de Guías', resultados)
       10. isLoading = false
```

**Mecanismo anti-duplicados:** Los `IdCarga` que reciben un 504 se registran en `idsEnProceso` (Set privado). Si el usuario vuelve a pulsar Generar sin refrescar, el componente bloquea el reintento de esos IDs mostrando un Swal de advertencia. El Set se limpia con cada llamada a `consultarDatos()`.

---

## Flujo principal: `generateValidaciones()`

```
generateValidaciones()
  1. validarSeleccion('validar')
  2. Extrae idsCargas únicos de filteredRows
  3. Para cada idCarga: POST validacionPedidos { IdCarga }
     → Concurrencia máxima: 3
  4. mostrarResumen('Validaciones', resultados)
```

---

## Métodos

### Públicos

| Método | Descripción |
|---|---|
| `ngOnInit()` | Carga los tipos de etiqueta disponibles al iniciar el componente |
| `consultarDatos()` | Verifica tiendas asignadas y dispara la carga de registros desde el backend |
| `fetchTamanios()` | Consulta y pobla `this.tamanos` con los tipos de etiqueta disponibles |
| `onCargaCompletaChange()` | Recalcula únicos de Registro, IdCarga y Tienda restringiéndolos a la carga seleccionada |
| `onRegistroChange()` | Recalcula únicos de IdCarga respetando los filtros de CargaCompleta y Registro activos |
| `onTiendaChange()` | Recalcula únicos de Registro e IdCarga respetando CargaCompleta y Tienda activos |
| `filterRows()` | Aplica todos los filtros activos sobre `rows` y actualiza `filteredRows`. Reinicia paginación a página 1 |
| `generateGuias()` | Punto de entrada para generar guías. Valida selección y pide confirmación si alarmas están desactivadas |
| `executeGuideGeneration()` | Ejecuta la generación masiva con concurrencia 2, anti-duplicados por 504 y resumen final |
| `generateValidaciones()` | Dispara validaciones masivas sobre los IdCarga filtrados con concurrencia 3 |
| `exportToCSV()` | Exporta `filteredRows` a un archivo `.csv` descargado localmente vía `file-saver` |
| `onToggleChange(event)` | Actualiza `toggleChecked` desde el evento del checkbox de alarmas |
| `getTotalByEstado()` | Retorna un string resumen con conteo por estado de los registros filtrados |
| `nextPage()` | Avanza una página si no es la última |
| `prevPage()` | Retrocede una página si no es la primera |

### Privados

| Método | Descripción |
|---|---|
| `fetchData()` | Consulta `CargaPedido`, descomprime (Zstd) y pobla `rows`, `columns` y los sets de filtros |
| `validarSeleccion(accion)` | Valida que haya CargaCompleta o IdCarga activo y mínimo 2 filtros. Retorna `boolean` |
| `refreshUniques(base)` | Recalcula `uniqueRegistros`, `uniqueIdCargas` y `uniqueTiendas` sobre el subconjunto dado |
| `extractUnique(data, key, sort?)` | Extrae valores únicos y no nulos de una propiedad. Opcionalmente los ordena alfabéticamente |
| `extractUniqueIds(data, key)` | Extrae valores únicos de una propiedad para construir la lista de IDs a procesar |
| `ejecutarConConcurrencia(tareas, limite)` | Ejecuta promesas en lotes del tamaño indicado, esperando cada lote antes del siguiente |
| `mostrarResumen(titulo, resultados)` | Muestra un Swal con tabla HTML coloreada por estado (OK/EN_PROCESO/ERROR) y conteos totales. Escapa HTML para prevenir XSS |

---

## Servicios utilizados

| Servicio | Métodos usados | Propósito |
|---|---|---|
| `ConsumoGenericoService` | `consultarGenerico()`, `insertarGenerico()` | Consulta y escritura a BD (CargaPedido, TipoEtiqueta, cargaPreenvios, validacionPedidos) |
| `DecompressionService` | `decompressZstd()`, `decompressGzip()` | Descomprime respuestas del backend (Zstd para CargaPedido, Gzip para TipoEtiqueta) |

---

## Endpoints que consume

| Método | Ruta | Cuándo |
|---|---|---|
| `GET` | `metodoGenerico?coleccion=TipoEtiqueta` | Al iniciar el componente (`ngOnInit`) |
| `GET` | `metodoGenerico?coleccion=CargaPedido` | Al pulsar "Consultar datos". Filtra por `Tienda` del sessionStorage |
| `POST` | `cargaPreenvios` | Por cada `IdCarga` al generar guías. Body: `{ IdCarga, TipoEtiqueta, CargarAlarmas }` |
| `POST` | `validacionPedidos` | Por cada `IdCarga` al validar pedidos. Body: `{ IdCarga }` |

---

## Estados de la vista

| Estado | Condición | Qué muestra |
|---|---|---|
| **Inicial** | `dataLoaded = false` | Panel de filtros vacíos, tabla oculta, botones deshabilitados |
| **Cargando** | `isLoading = true` | Controles deshabilitados. Spinner / feedback visual |
| **Con datos** | `dataLoaded = true`, `filteredRows.length > 0` | Tabla paginada con registros y resumen de estados |
| **Sin resultados** | `filteredRows.length === 0` tras filtrar | Tabla vacía (sin registros que coincidan) |
| **Proceso en curso** | IdCarga en `idsEnProceso` | Swal de bloqueo si el usuario intenta volver a generar sin refrescar |
| **Error** | Catch en `fetchData` | `console.error`. Sin Swal visible (error silencioso en carga de datos) |

---

## Filtros encadenados

Los selectores de filtro son dependientes: al cambiar un filtro superior, los filtros inferiores se recalculan para mostrar solo los valores presentes en el subconjunto actual.

```
IdCargaCompleta (global, siempre todos los valores)
  └─> Tienda (se restringe a la carga seleccionada)
        └─> Registro (se restringe a carga + tienda)
              └─> IdCarga (se restringe a carga + tienda + registro)

Estado (siempre global, no afectado por la cadena)
```

Al cambiar `CargaCompleta`: resetea Registro, IdCarga y recalcula Tienda.
Al cambiar `Tienda`: resetea Registro, IdCarga y los recalcula.
Al cambiar `Registro`: resetea IdCarga y lo recalcula.

---

## Lógica de generación — protección contra duplicados

El componente protege contra la duplicación de guías con dos mecanismos:

1. **Filtro por estado antes de enviar:** Solo se envían al backend filas con `Estado = 'Pendiente'` o (`Estado = 'Cargado'` y sin `NumeroPreenvio`). Las filas `Cargado` con preenvío asignado se excluyen automáticamente.
2. **Set `idsEnProceso`:** Si el backend responde con un 504 (timeout, el proceso sigue corriendo en plataforma), el `IdCarga` se registra en el Set. Cualquier intento posterior de generar sin refrescar los datos es bloqueado con un Swal de advertencia. El Set se vacía al llamar a `consultarDatos()`.

---

## Columnas fijas (`COLUMNAS_FIJAS`)

La tabla no genera columnas dinámicamente desde las claves del objeto; en cambio, usa una lista fija de 60+ columnas predefinidas. Esto garantiza un orden consistente independientemente de los datos recibidos.

Incluye entre otras: `Fecha Carga`, `IdCarga`, `Estado`, `IdTienda`, `Tienda`, datos del destinatario, transportadora, dimensiones y peso, forma de pago, stocks (hasta 12 productos), `NumeroPreenvio`, totales de flete/recaudo, `ERROR`, `IdCargaCompleta`, `Registro`.

---

## Navegación desde esta vista

Esta vista no realiza redirecciones. Es un componente autocontenido del módulo de pedidos masivo.

---

## Changelog de la vista

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-03-31 | Iker | Mejoras en rendimiento y módulo pedidos masivo |
| Anterior | Iker | Modularización del código del módulo pedidos masivo |

---

## Observaciones

- **Columnas fijas vs dinámicas:** A diferencia de `ExportaGuiasComponent` (que genera columnas dinámicamente desde las claves del objeto), este componente usa `COLUMNAS_FIJAS` para garantizar orden y consistencia de la tabla. Si el backend añade nuevas columnas, no aparecerán automáticamente en la tabla; hay que agregarlas a `COLUMNAS_FIJAS`.
- **Concurrencia 2 en generación:** La generación de guías envía máximo 2 `IdCarga` en paralelo. Es más conservadora que la validación (concurrencia 3) por el mayor peso de la operación en el backend.
- **504 es éxito diferido:** Un HTTP 504 en `cargaPreenvios` no indica fallo; el backend sigue procesando de forma asíncrona. El componente lo trata como `EN_PROCESO` y lo muestra en amarillo en el resumen.
- **Toggle de alarmas:** Cuando el toggle está desactivado (`toggleChecked = false`), se envía `CargarAlarmas: false` al backend. El componente pide confirmación explícita al usuario antes de proceder sin alarmas.
- **Estimación de tiempo:** Para lotes grandes (> 50 pedidos), el componente calcula `Math.ceil(pendientes / 60)` minutos como estimación orientativa para el usuario, asumiendo ~60 pedidos por minuto en el backend.
- **Exportación CSV:** `exportToCSV()` usa la librería `file-saver` y genera el archivo en memoria sin llamadas al backend. Las celdas se serializan con `JSON.stringify` para manejar valores con comas.
- **Sin manejo de error visible en `fetchData`:** Los errores en la carga de datos solo se loggean en consola (`console.error`). El usuario no recibe feedback visual si la consulta inicial falla.
