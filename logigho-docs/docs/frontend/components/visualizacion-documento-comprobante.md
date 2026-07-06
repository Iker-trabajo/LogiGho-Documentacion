---

## Autor: Adalberto González

Fecha creacion: 2026-06-16  
Estado: desarrollo  
Tipo: componente  

# Componente: VisualizacionDocumentoComprobanteComponent

**Selector:** `app-visualizacion-documento-comprobante`  
**Ubicación:** `src/app/components/visualizacion-documento-comprobante`  
**Acceso:** Autenticado | usado desde `app-tables` vía creación dinámica

---

## ¿Qué hace?

Ventana para visualizar y descargar comprobantes de pago o documentos de las tiendas. 
Permite previsualizar archivos PDF e imágenes, descargarlos con un nombre legible y acceder a funcionalidades adicionales como zoom, pantalla completa y cierre mediante teclado.

---

## Decoradores

| Decorador | Nombre | Tipo | Descripción |
| --- | --- | --- | --- |
| `@Input` | `transactionId` | `string` | `_id` de la transacción — también es la key del archivo en S3 |
| `@Input` | `isOpen` | `boolean` | Controla la visibilidad del modal en el DOM |
| `@Input` | `opcion` | `string` | `'1'` = bucket comprobantes de pago · `'2'` = bucket documentos tienda |
| `@Input` | `rowData` | `Record<string, any>` | Datos del row de la tabla, usados para componer el nombre de descarga |
| `@Output` | `closeEvent` | `EventEmitter<void>` | Emite cuando el modal se cierra |

---

## Propiedades clave

| Propiedad | Tipo | Descripción |
| --- | --- | --- |
| `fileUrl` | `string` | Blob URL generada con `URL.createObjectURL` — se pasa al `<iframe>` o `<img>` via `SafeUrlPipe` |
| `isLoading` | `boolean` | Controla el spinner; se pone en `false` tras recibir respuesta de S3 |
| `isPdf` / `isImage` | `boolean` | El template los usa para decidir qué elemento renderizar |
| `fileBlob` | `Blob` | Archivo en memoria — necesario para el botón de descarga |

---

## Servicios y endpoints

| Servicio | Endpoint | Cuándo |
| --- | --- | --- |
| `GetObjectService` | `obtenerObjeto(bucket, key)` | `ngOnInit` — descarga el archivo de S3 según `opcion` |

---

## Flujo principal

```
ngOnInit()
  -> cargarDesdeS3(bucket)
      -> GetObjectService.obtenerObjeto()
      -> base64ToBlob()         // parsea "mimeType|ext|base64" → Blob
      -> URL.createObjectURL()  // genera blob URL para el iframe
      -> isLoading = false
      -> cdr.detectChanges()    // forzado porque la respuesta llega en un Observable

downloadFile()
  -> buildFileName()            // compone "comprobante_{Fecha}_{Tienda}.{ext}"
  -> URL.createObjectURL()      // nuevo objeto URL para descarga
```

---

## Historial de cambios

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-06-16 | Adalberto González | Implementación y refactor del modal de visualización de documentos: rediseño de interfaz, mejoras en visualización y descarga de archivos, optimización de la carga desde S3 y centralización de configuraciones y dependencias. |


---

## Observaciones

- El modal se abre dinámicamente desde `tables.component.ts` con `modalContainer.createComponent()`.
