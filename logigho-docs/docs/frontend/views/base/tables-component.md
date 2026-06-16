---

## Autor: Adalberto González

Fecha creacion: 2026-06-16  
Estado: desarrollo  
Tipo: componente  

# Componente: TablesComponent

**Selector:** `app-tables`  
**Ubicación:** `src/app/views/base/tables`  
**Acceso:** Autenticado | componente genérico reutilizable en toda la plataforma

---

## ¿Qué hace?

Tabla reutilizable utilizada como estándar para la plataforma para la visualización y gestión de datos. 
Permite realizar búsquedas, ordenar información, gestionar la visibilidad de columnas, navegar mediante paginación y ejecutar acciones específicas mediante la apertura dinámica de modales.

---

## Decoradores

| Decorador | Nombre | Tipo | Descripción |
| --- | --- | --- | --- |
| `@Input` | `columns` | `TablaColumn[]` | Definición de columnas: `{ key, title }` |
| `@Input` | `rows` | `TablaRow[]` | Datos a mostrar; cada row debe incluir `_id` si abre modales |
| `@Input` | `nombre` | `string` | Identificador de la tabla — parte de la clave de `localStorage` |
| `@Input` | `descripcion` | `string` | Segunda parte de la clave de `localStorage` |
| `@Input` | `nombreModal` | `string` | Nombre del componente modal a crear dinámicamente al pulsar acción |
| `@Input` | `actions` | `boolean` | Muestra el botón de acción definido por `valueactions` |
| `@Input` | `valueactions` | `string` | Etiqueta del botón de acción (ej: `'Ver Comprobante'`) |
| `@Input` | `columnTitles` | `Record<string, string>` | Títulos personalizados que sobreescriben los de `columns[].title` |
| `@Input` | `filtroEstado` | `string[]` | Filtra las filas mostrando solo las que coincidan con algún estado de la lista |
| `@Output` | `closeEvent` | `EventEmitter<void>` | Re-emite el cierre del modal hacia el componente padre |
| `@Output` | `updated` | `EventEmitter<void>` | Emite cuando un modal hijo confirma una actualización |

---

## Propiedades clave

| Propiedad | Tipo | Descripción |
| --- | --- | --- |
| `visibleColumns` | `string[]` | Keys de columnas actualmente visibles; persiste en `localStorage` con clave `visibleColumns_{nombre}` |
| `modalContainer` | `ViewContainerRef` | Contenedor donde se crean los modales dinámicamente con `createComponent()` |
| `filterSubject` | `Subject<void>` | Debounce de 300ms para la búsqueda — evita filtrar en cada tecla |
| `itemsPerPage` | `number` | Filas por página; por defecto `50` |

---

## localStorage

El componente persiste dos preferencias por tabla, usando `nombre` y `descripcion` como parte de la clave:

| Clave | Contenido |
| --- | --- |
| `visibleColumns_{nombre}` | Array de keys de columnas visibles |
| `columnOrder_{descripcion}_{nombre}` | Array de keys en el orden actual |

> Si se cambian las columnas desde el padre (nuevo `@Input columns`), `applyColumnTitles()` actualiza el `localStorage` para evitar que columnas nuevas queden ocultas por caché stale.

---

## Modales dinámicos

Al pulsar el botón de acción, `abrirModalDocumentos(row)` crea el componente correspondiente según `nombreModal`:

| `nombreModal` | Componente creado |
| --- | --- |
| `'VisualizacionDocumentoComprobanteComponent'` | `VisualizacionDocumentoComprobanteComponent` — pasa `transactionId = row['_id']` y `rowData = row` |
| *(otros)* | `AprobacionTransaccionComponent`, `GestionNovedadPedidoComponent`, etc. |

---

## Flujo principal

```
ngOnChanges(columns / columnTitles)
  -> loadColumnOrder()
  -> loadColumnPreferences()
  -> applyColumnTitles()     // sobreescribe titles y sincroniza localStorage

abrirModalDocumentos(row)
  -> modalContainer.clear()
  -> createComponent(VisualizacionDocumentoComprobanteComponent)
  -> instance.transactionId = row['_id']
  -> instance.rowData       = row
```

---

## Historial de cambios

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-06-16 | Adalberto González | Mejoras en la configuración de la tabla: incorporación de títulos de columnas personalizados y ajuste de la apertura de modales para soportar nombres de descarga más descriptivos en comprobantes y documentos de tienda. |

---

## Observaciones

