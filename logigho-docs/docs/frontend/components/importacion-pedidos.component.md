---

## Autor: Adaberto González

Fecha creacion: 2026-06-12  
Estado: desarrollo  
Tipo: componente

# Componente: ImportacionPedidosComponent

**Selector:** `app-importacion-pedidos`  
**Ubicación:** `src/app/components/importacion-pedidos/importacion-pedidos`  
**Acceso:** Autenticado | usado desde el vista de pedidos de el modulo tienda

---

## ¿Qué hace?

Este módulo permite realizar la carga de varios pedidos a partir de un archivo de Excel. Durante el proceso, la información es validada para identificar posibles registros duplicados y asegurar la consistencia de los datos antes de ser incorporados al sistema.

---

## Decoradores

| Decorador | Nombre            | Tipo                 | Descripción                                                  |
| --------- | ----------------- | -------------------- | ------------------------------------------------------------ |
| `@Input`  | `isOpen`          | `boolean`            | Controla si el modal está visible                            |
| `@Input`  | `tiendas`         | `any[]`              | Lista de tiendas del usuario; se ordena alfabéticamente al recibirse |
| `@Output` | `closeEvent`      | `EventEmitter<void>` | Emitido al cerrar el modal                                   |
| `@Output` | `refreshDataEvent`| `EventEmitter<void>` | Emitido para solicitar recarga de datos al padre             |
| `@ViewChild` | `archivo`      | `ElementRef`         | Referencia al input file para poder resetearlo programáticamente |

---

## Propiedades clave

| Propiedad               | Tipo          | Descripción                                                                          |
| ----------------------- | ------------- | ------------------------------------------------------------------------------------ |
| `rowsMemory`            | `TablaRow[]`  | Caché en memoria de todos los registros de `CargaPedido`; se usa para calcular `IdCarga`, `IdOrdenPedido` y detectar duplicados contra BD |
| `isCargaPedidoModalOpen`| `boolean`     | Controla la apertura del modal hijo `ModalTableComponent` tras una carga exitosa     |
| `isButtonDisabled`      | `boolean`     | Bloquea el botón Importar durante el procesamiento para evitar doble envío           |
| `fileToUpload`          | `File \| null`| Archivo Excel seleccionado por el usuario, listo para procesarse                     |
| `tiendaSeleccionadaNombre` | `string`   | Nombre visible en el buscador dinámico de tiendas (separado del ID guardado en el form) |

---

## Servicios y endpoints

| Servicio                  | Método                      | Endpoint                                                        | Cuándo                                          |
| ------------------------- | --------------------------- | --------------------------------------------------------------- | ----------------------------------------------- |
| `GetObjectService`        | `obtenerObjeto`             | S3 bucket `logigho-plantillas` / `massive-order.xlsx`           | Al abrir el modal — descarga la plantilla Excel |
| `ConsumoGenericoService`  | `consultarGenerico`         | `metodoGenerico?coleccion=CargaPedido` (compresión Zstd)        | `fetchTableData` — carga histórico de pedidos   |
| `ConsumoGenericoService`  | `consultarGenericoAnidado`  | `metodoGenerico?coleccion=Productos&perfilproducto=Publico&Tienda=...` | Solo tiendas Dropshipping — valida stock    |
| `ConsumoGenericoService`  | `insertarGenerico`          | `cargaPedidos`                                                  | Al confirmar la carga del Excel                 |
| `DecompressionService`    | `decompressZstd`            | —                                                               | Descomprimir respuesta de `CargaPedido`         |
| `DecompressionService`    | `decompressGzip`            | —                                                               | Descomprimir respuesta de `Productos` (Dropshipping) |

---

## Flujo principal

```
ngOnChanges(isOpen = true)
  -> obtenerArchivo()        // descarga plantilla desde S3
  -> fetchTableData()        // carga CargaPedido en rowsMemory y abre modal hijo

uploadFile()  [8 etapas]
  1. Validar tienda seleccionada (form.nombreTienda)
  2. Bloquear si TipoTienda === solo 'Proveedor'
  3. Leer archivo con FileReader (readAsBinaryString)
  4. Parsear Excel con XLSX → headers + filas
  5. Refrescar rowsMemory via fetchTableData(false) para tener datos actualizados
     -> calcular newIdCarga = max(IdCarga) + 1
     -> construir existingKeys con getUniqueKey() sobre rowsMemory
  6. Construir jsonData:
     -> asignar campos de sistema (FechaCarga, IdCarga, Estado, IdTienda, Tienda)
     -> asignar IdOrdenPedido incremental
     -> filtrar filas vacías
     -> deduplicar dentro del Excel (seenKeys)
     -> deduplicar contra BD (existingKeys)
  7. Si hay duplicados (excel o BD) → Swal warning con resumen
     -> usuario puede continuar con nuevos o cancelar
  8. Si tienda es Dropshipping → validar stock de productos
     -> si hay inválidos → Swal error y return
  9. insertarGenerico(jsonData, "cargaPedidos")
     -> éxito: Swal success + fetchTableData() + close()
     -> error: Swal error + resetFileInput()

getUniqueKey(record)
  -> concatena solo campos del Excel (excluye camposSistema)
  -> garantiza que el mismo pedido de dos cargas distintas produzca la misma clave
```

---

## Historial de cambios

| Fecha      | Autor             | Cambio                                                                                     |
| ---------- | ----------------- | ------------------------------------------------------------------------------------------ |
| 2026-06-12 | Adaberto González | Ajustes al proceso de importación de pedidos para mejorar la validación de datos, evitar registros duplicados y optimizar la experiencia de usuario durante la carga y consulta de información              |

---

## Observaciones

> Deuda técnica, comportamientos no obvios, decisiones de diseño que no se ven en el código.

- `refreshDataEvent` está declarado pero no se emite en ningún flujo; el padre no lo necesita hoy porque `fetchTableData` recarga internamente.
