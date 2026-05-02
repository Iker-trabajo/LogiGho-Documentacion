---
autor: Adalberto González
fecha_creacion: 2026-04-29
ultima_actualizacion: 2026-04-29
estado: desarrollo
nivel: 2
---

# Conciliación de Pagos

**Autor:** Adalberto González

**Selector:** `app-conciliacionpagos`

**Ubicación:** `SitioLogiGho/src/app/views/controller/conciliacion-pagos`

---

## ¿Qué hace?

Centraliza los pagos realizados por todas las transportadoras integradas al sistema (Interrapidísimo, Envia, X-Cargo, Servientrega, D2E). Permite consultar los registros almacenados en MongoDB, buscar por cualquier campo, crear o modificar registros, importar nuevos pagos desde Excel y exportar los datos actuales a Excel.

---

## Roles y acceso

| Acceso | Descripción |
|---|---|
| Autenticado | Necesita login |
| Rol específico | Requiere rol: `controller` |

---

## Estructura de archivos

```
conciliacion-pagos/
├── conciliacionpagos.component.ts
├── conciliacionpagos.component.html
├── conciliacionpagos.component.scss
├── conciliacionpagos.component.spec.ts
└── strategies/
    ├── transportadora-pago.strategy.ts    ← Contrato/interfaz base
    ├── transportadora-pago.factory.ts     ← Registro central de transportadoras
    └── transportadoras/
        └── d2e.strategy.ts                ← Implementación D2E
```

---

## Estado de implementación por transportadora

| Transportadora | Colección MongoDB | Patrón | Componente de importación |
|---|---|---|---|
| Interrapidísimo | `ConciliacionPagos` | Legacy | `app-importacion-pagos` |
| Envia | `ConciliacionPagosEnvia` | Legacy | `app-importacion-pagos-envia` |
| X-Cargo | `ConciliacionPagosXcargo` | Legacy | `app-importacion-pagos-xcargo` |
| Servientrega | `ConciliacionPagosServientrega` | Legacy | `app-importacion-pagos-servientrega` |
| D2E | `ConciliacionPagosD2E` | **Strategy** | `app-importacion-pagos-generico` |

> Las transportadoras legacy funcionan correctamente. La migración al patrón Strategy queda pendiente de evaluación con el equipo, hacer gradualmente e ir probando en producción para no afectar por completo al sistema.

---

## Propiedades del componente

### Legacy (por transportadora)

Cada transportadora legacy mantiene sus propias variables de estado. Los nombres siguen el patrón `rows*`, `rowsMemory*`, `columns*`, `searchValue*`:

| Transportadora | Filas | Copia memoria | Columnas | Búsqueda |
|---|---|---|---|---|
| Inter | `rows` | `rowsMemory` | `columns` | `searchValue` |
| Envia | `rowsEnvia` | `rowsMemoryEnvia` | `columnsEnvia` | `searchValueEnvia` |
| X-Cargo | `rowsXcargo` | `rowsMemoryXcargo` | `columnsXcargo` | `searchValueXcargo` |
| Servientrega | `rowsServientrega` | `rowsMemoryServientrega` | `columnsServientrega` | `searchValueServientrega` |

### Strategy (D2E y futuras)

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `tablaEstadoMap` | `Map<string, TablaEstado>` | `new Map()` | Estado centralizado de todas las transportadoras Strategy |
| `d2eStrategy` | `TransportadoraPagoStrategy` | `Factory.crear('d2e')` | Instancia de la strategy D2E |
| `isD2EModalOpen` | `boolean` | `false` | Visibilidad del modal de importación D2E |

### Campos de formularios (Legacy)

| Transportadora | Campos |
|---|---|
| Inter | `IdCargaPago`, `Guias`, `Fecha_venta`, `Valor_total`, `Nombre_destinatario`, `Conciliacion`, `Fecha`, `Valor`, `Banco`, `Observacion` |
| Envia | `IdCargaPago`, `Num_Traslado`, `Fec_Traslado`, `Guia`, `Num_Documentos`, `Valor_Producto`, `Nom_Destinatario`, `CiudadDestino`, `Tel_Destinatario` |
| X-Cargo | `IdCargaPago`, `Fecha_consignacion`, `Guiaxcargo`, `Valor_unidad`, `Monto_confirmado`, `Tipo_recaudo` |

---

## Métodos

### Carga de datos

#### Legacy: `fetchTableData()`, `fetchTableDataEnvia()`, `fetchTableDataXcargo()`, `fetchTableDataServientrega()`

Cada método carga la tabla de su transportadora. El proceso es idéntico en todos:
1. Llama a `consultarGenerico` con la colección correspondiente
2. Descomprime con `DecompressionService`
3. Aplana y sanitiza los items
4. Genera columnas dinámicas excluyendo `_id` e `IdCargaPago`
5. Asigna filas y copia a `rowsMemory*` para restaurar filtros

#### Strategy: `fetchTableDataByStrategy(strategy)`

Mismo proceso que los legacy pero genérico. Usa `strategy.coleccion` y `strategy.sanitize(item)`. Almacena el resultado en `tablaEstadoMap`.

---

### Cálculo de totales

#### Legacy: `getTotalByCarga*()` / `getTotalByMonto*()`

Cada transportadora tiene dos métodos: uno cuenta filas (`rows*.length`) y otro suma el campo de monto (`Valor_total`, `Valor_Producto`, `Valor_unidad`, `VALOR_MOVILIZADO` según la transportadora).

#### Strategy: `getTotalMontoByStrategy(strategy)`

Suma el campo definido en `strategy.campoTotal` de todas las filas de la transportadora.

---

### Crear / modificar registros (Legacy)

#### `onCreateOrModify()`, `onCreateOrModifyEnvia()`, `onCreateOrModifyXcargo()`

Cada método valida los campos requeridos, llama a `insertarGenerico` con la colección correspondiente y recarga la tabla. Las transportadoras Strategy usan `app-importacion-pagos-generico` para esta funcionalidad.

---

### Filtrado

#### Legacy: `filterPagos()`, `filterPagosEnvia()`, `filterPagosXcargo()`, `filterPagosServientrega()`

Filtran las filas de su transportadora comparando el JSON de cada row contra el valor de búsqueda (case-insensitive). Si el valor está vacío, restauran desde `rowsMemory*`.

#### Strategy: `filterByStrategy(strategy, valorBuscado)`

Mismo comportamiento pero usando `tablaEstadoMap` para leer y escribir el estado.

---

### Exportación

#### Legacy: `exportInter()`, `exportEnvia()`, `exportXcargo()`, `exportServientrega()`

Cada método llama a `exportToExcel()` con las filas y nombre de archivo de su transportadora.

#### Strategy: `exportarByStrategy(strategy)`

Obtiene las filas con `getRowsByStrategy(strategy)` y llama a `exportToExcel()` con `strategy.exportFileName`.

#### `exportToExcel(data, fileName)`

Convierte el array a worksheet con SheetJS, crea el workbook y descarga el archivo con `file-saver`.

---

### Auxiliares de estado (Strategy)

| Método | Descripción |
|---|---|
| `getStateByStrategy(strategy)` | Retorna `{ rows, rowsMemory, columns, valorBuscado }` desde `tablaEstadoMap` |
| `getRowsByStrategy(strategy)` | Retorna las filas visibles actuales |
| `getColumnsByStrategy(strategy)` | Retorna las columnas generadas |

---

### Auxiliares generales

| Método | Descripción |
|---|---|
| `generateColumns(data)` | Genera columnas excluyendo `_id` e `IdCargaPago`, capitaliza títulos |
| `generateRows(data)` | Transforma arrays a strings separados por coma y objetos anidados a "clave: valor" |
| `capitalizeFirstLetter(string)` | Capitaliza la primera letra de una cadena |

---

### Control de modales

Cada transportadora tiene `open*Modal()` y `close*Modal()`. Los nombres siguen el patrón:

| Transportadora | Abrir | Cerrar |
|---|---|---|
| Inter | `openImportPagoModal()` | `closeImpPagoModal()` |
| Envia | `openImportPagoEnviaModal()` | `closeImpPagoEnviaModal()` |
| X-Cargo | `openImportPagoXcargoModal()` | `closeImpPagoXcargoModal()` |
| Servientrega | `openImportPagoServientregaModal()` | `closeImpPagoServientregaModal()` |
| D2E | `openD2EModal()` | `closeD2EModal()` |

---

### Ciclo de vida

#### `ngOnInit()`

```
ngOnInit()
  ├── fetchTableData()                    → carga Inter (legacy)
  ├── fetchTableDataEnvia()               → carga Envia (legacy)
  ├── fetchTableDataXcargo()              → carga X-Cargo (legacy)
  ├── fetchTableDataServientrega()        → carga Servientrega (legacy)
  └── TransportadoraPagoFactory.obtenerTodas().forEach(strategy =>
        fetchTableDataByStrategy(strategy))  → carga D2E y futuras
```

---

## Servicios Angular utilizados

| Servicio | Métodos usados | Propósito |
|---|---|---|
| `ConsumoGenericoService` | `consultarGenerico()`, `insertarGenerico()` | Lectura y escritura en MongoDB |
| `DecompressionService` | `decompressGzip()` | Descompresión de datos paginados del API |

---

## Subcomponentes

| Componente | Selector | Descripción |
|---|---|---|
| `ImportacionPagosComponent` | `app-importacion-pagos` | Modal importación Inter (legacy) |
| `ImportacionPagosEnviaComponent` | `app-importacion-pagos-envia` | Modal importación Envia (legacy) |
| `ImportacionPagosXcargoComponent` | `app-importacion-pagos-xcargo` | Modal importación X-Cargo (legacy) |
| `ImportacionPagosServientregaComponent` | `app-importacion-pagos-servientrega` | Modal importación Servientrega (legacy) |
| `ImportacionPagosGenericoComponent` | `app-importacion-pagos-generico` | Modal importación genérico — recibe `TransportadoraPagoStrategy` como `@Input()` |
| `TablesComponent` | `app-tables` | Tabla reutilizable con columnas dinámicas |

---

## Estilos

| Elemento | Valor |
|---|---|
| Gradiente principal | `#121B60 → #3d4fd6` — botones y encabezado |
| Fondo totales | `#f8f9fa` — barra inferior de totales |
| Animación entrada | `fadeInLeft` 0.5s |
| Transiciones modales | 0.3s |

---

## Changelog

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-04-29 | Adalberto González | Implementación de patrón Strategy para D2E. Creación de `TransportadoraPagoStrategy`, `TransportadoraPagoFactory` y `D2EStrategy`. Creación de `app-importacion-pagos-generico`. Métodos genéricos: `fetchTableDataByStrategy()`, `getStateByStrategy()`, `filterByStrategy()`, `getTotalMontoByStrategy()`, `exportarByStrategy()`. |

---

## Observaciones

- Las transportadoras legacy mantienen variables individuales de estado (`rows`, `rowsEnvia`, etc.). El `tablaEstadoMap` centraliza el estado solo de las que usan Strategy.
- Los métodos `onCreateOrModify*()` son exclusivos de las legacy. Las Strategy usan `app-importacion-pagos-generico`.
- La colección `ConciliacionPagosD2E` define su schema con la primera inserción; los campos están en `D2EStrategy.columnMapping`.
- La búsqueda global compara el JSON completo del registro (todos los campos) contra el texto ingresado.
