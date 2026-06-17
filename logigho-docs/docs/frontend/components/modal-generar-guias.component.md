---

## Autor: Adaberto González

Fecha creacion: 2026-06-12  
Estado: desarrollo  
Tipo: componente

# Componente: ModalTableComponent

**Selector:** `app-modal-table`  
**Ubicación:** `src/app/components/modal-generar-guias/modal-generar-guias`  
**Acceso:** Autenticado | usado desde el vista de pedidos de el modulo tienda

---

## ¿Qué hace?

Este módulo permite consultar el historial de pedidos cargados en el sistema mediante una tabla que puede filtrarse según diferentes criterios de búsqueda.

Desde esta pantalla, los operadores pueden seleccionar una carga específica y ejecutar procesos relacionados con el despacho de los pedidos, como la generación de guías de envío.

---

## Decoradores

| Decorador | Nombre           | Tipo                   | Descripción                                              |
| --------- | ---------------- | ---------------------- | -------------------------------------------------------- |
| `@Input`  | `columns`        | `TablaColumn[]`        | Definición de columnas para la tabla                     |
| `@Input`  | `rows`           | `TablaRow[]`           | Filas con los registros de `CargaPedido`                 |
| `@Input`  | `filtroEstado`   | `string[]`             | Estados a resaltar visualmente en la tabla               |
| `@Input`  | `isOpen`         | `boolean`              | Controla si el modal está visible                        |
| `@Input`  | `totalRegistros` | `number`               | Total de registros de la carga activa                    |
| `@Output` | `closeEvent`     | `EventEmitter<void>`   | Emitido al cerrar el modal                               |
| `@Output` | `refreshDataEvent` | `EventEmitter<void>` | Emitido para solicitar recarga de datos al padre         |

---

## Propiedades clave

| Propiedad       | Tipo          | Descripción                                                                 |
| --------------- | ------------- | --------------------------------------------------------------------------- |
| `filteredRows`  | `TablaRow[]`  | Subconjunto de `rows` después de aplicar los filtros activos                |
| `tamanios`      | `any[]`       | Opciones de tamaño de etiqueta cargadas desde la colección `TipoEtiqueta`   |
| `toggleChecked` | `boolean`     | Indica si se deben cargar alarmas al generar las guías (default: `true`)    |
| `isGenerating`  | `boolean`     | Flag anti-doble ejecución: bloquea una segunda llamada a `cargaPreenvios`   |
| `selectedCarga` | `string`      | Id de la carga seleccionada en el filtro; `"0"` o vacío significa "Todos"   |

---

## Servicios y endpoints

| Servicio                  | Método              | Endpoint                                    | Cuándo                          |
| ------------------------- | ------------------- | ------------------------------------------- | ------------------------------- |
| `ConsumoGenericoService`  | `consultarGenerico` | `metodoGenerico?coleccion=TipoEtiqueta`     | `ngOnInit` → cargar tamaños     |
| `ConsumoGenericoService`  | `insertarGenerico`  | `cargaPreenvios`                            | Al confirmar "Generar Guías"    |
| `ConsumoGenericoService`  | `insertarGenerico`  | `validacionPedidos`                         | Al pulsar "Ejecutar Validaciones"|
| `DecompressionService`    | `decompressGzip`    | —                                           | Descomprimir respuesta de tamaños|

---

## Flujo principal

```
ngOnInit()
  -> filteredRows = rows (sin filtro)
  -> fetchTableDataTamanios()
       -> consultarGenerico("TipoEtiqueta")
       -> decompressGzip(data.Resultado)
       -> tamanios = resultado mapeado

ngOnChanges(isOpen = true)
  -> filterRows()  // re-aplica filtros al reabrir

filterRows()
  -> parte de rows completo
  -> aplica selectedCarga  (si != "0")
  -> aplica selectedEstado (si != "0")
  -> aplica selectedSize   (si != "0")
  -> actualiza filteredRows

generateGuides()
  -> si toggleChecked=false  -> confirmación Swal
  -> executeGuideGeneration()
       -> guard: isGenerating → alerta y return
       -> evalúa todosCargados solo sobre pedidos de selectedCarga
       -> si todosCargados → alerta info y return
       -> si !selectedSize  → alerta error y return
       -> insertarGenerico({ IdCarga, TipoEtiqueta, CargarAlarmas }, "cargaPreenvios")
       -> isGenerating = true (se libera en next/error)
       -> Swal info + close()

generateValidaciones()
  -> insertarGenerico({ IdCarga }, "validacionPedidos")  [fire-and-forget]
  -> Swal info + close()

exportToCSV()
  -> serializa filteredRows con columnas visibles
  -> convierte a Windows-1252 (ANSI) para compatibilidad con Excel
  -> saveAs blob "table-data.csv"
```

---

## Historial de cambios

| Fecha      | Autor              | Cambio                                                                           |
| ---------- | ------------------ | -------------------------------------------------------------------------------- |
| 2026-06-12 | Adaberto González  | Correcciones y mejoras en el proceso de generación de guías, optimizando la validación de pedidos cargados, el funcionamiento de los filtros, el manejo de operaciones asíncronas y la limpieza de código no utilizado para mejorar la estabilidad y mantenibilidad del módulo.           |

---

## Observaciones

> Deuda técnica, comportamientos no obvios, decisiones de diseño que no se ven en el código.

- `cargaPreenvios` y `validacionPedidos` son Lambdas asíncronas en producción: De las cuales actualmente esta en funcionamiento solo `cargaPreenvios`.
- `refreshDataEvent` está declarado pero no se emite en ningún flujo actual; el padre debe escucharlo si necesita recargar `CargaPedido` tras la generación.
- La exportación CSV usa codificación Windows-1252 manual para preservar tildes y ñ al abrir en Excel en Windows.
