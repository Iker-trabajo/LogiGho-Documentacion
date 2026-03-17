---
autor: Iker Acevedo Vargas
fecha_creacion: 2026-03-16
ultima_actualizacion: 2026-03-16
estado: desarrollo
nivel: 3
---

# Vista: Documentos de Tiendas LogiGho

**Selector:** `app-documentos-tiendas`

**Ubicación:** `src/app/views/documentos-tiendas`

---

## **¿Qué hace?**

Permite a los roles administradores gestionar los documentos legales requeridos por cada tienda de LogiGho cada tienda debe de tener: Documento de identificación, RUT, Certificado Bancario, Contrato de Mandato, luego el administrador debe revisar los documentos de cada usuario, para después validar cada uno, con eso las tiendas se agrupan como completado. Además, cada tienda puede tener una autorización estas autorizaciones le permiten a cada tienda retirar dinero de la wallet a personas diferentes del propietario de la tienda. Cada autorización para marcar como completo debe tener; documento de identidad, RUT actualizado, certificacion bancaria y carta de autorización validados.

---

## **Propiedades principales**

| Propiedad | Tipo | Descripción |
| --- | --- | --- |
| `tiendas` | `TiendaDocumentStatus[]` | Lista completa cargada al iniciar |
| `tiendasFiltradas` | `TiendaDocumentStatus[]` | Subconjunto activo para aplicar filtros |
| `tiendaSeleccionada` | `TiendaDocumentStatus | null` | Tienda abierta en el modal de detalle |
| `autActiva` | `AutorizacionInfo | null` | Autorización activa dentro del modal de detalle |
| `tabActiva` | `string` | `tienda` o el `id` de una autorización |
| `autorizacionParaSubir` | `AutorizacionInfo | null` | Destino al abrir el modal de subir; `null` si es para la tienda |
| `docTransactionId` | `string` | S3 key del documento a visualizar |

> El resto de propiedades son flags de control de modales y campos de formulario cuyo nombre es autoexplicativo, ademas todo esta docuemntado en el codigo tambien.
> 

---

## **Servicios y endpoints**

| Servicio | Método | Endpoint | Cuándo |
| --- | --- | --- | --- |
| `TiendasService` | `consultarTienda()` | GET interno para traer todas las tiendas | Al inicializar |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET metodoGenerico?coleccion=TiendaDocuments` | Al inicializar |
| `ConsumoGenericoService` | `insertarGenerico()` | `POST metodoGenerico?coleccion=TiendaDocuments` | Al subir un documento |
| `ConsumoGenericoService` | `actualizarGenerico()` | `PUT metodoGenerico?coleccion=TiendaDocuments` | Al validar/invalidar o eliminar un documento |
| `ConsumoGenericoService` | `eliminarGenerico()` | `DELETE metodoGenerico?coleccion=TiendaDocuments&_id={id}` | Al eliminar una autorización completa |
| `PutObjectService` | `cargarInformacion()` | PUT S3 `logigho-documentos-tienda-prod` | Al subir archivo |
| `GetObjectService` | `obtenerObjeto()` | GET S3 `logigho-documentos-tienda-prod` | Al descargar archivo |

---

## **Observaciones**

- **Eliminación no borra S3:** Al eliminar un documento solo elimina de MongoDB, el archivo físico en el bucket no se elimina. Ya que esto nos permite a nosotros como desarrolladores tener un backup y no depender solo de la base de datos.
- **UUID de autorización generado en cliente:** El `IdAutorizacion` se genera con `auth_${Date.now()}_${random}` en el frontend al subir el primer documento. Si el usuario cierra el modal antes de subir, la autorización provisional se descarta sin rastro.
- **SSR seguro:** `ngOnInit` verifica `isPlatformBrowser()` antes de hacer cualquier llamada.

---

## **Secciones de la vista**

| # | Sección | Descripción |
| --- | --- | --- |
| 1 | **Filtros y tabs** | Búsqueda por texto, tabs de cumplimiento (Todos / Incompleto / Pendiente / Completo), selector de estado operativo y botón limpiar |
| 2 | **Tabla de tiendas** | Lista paginada con columnas: ID, nombre, ecosistema, fulfillment, estado, fecha contrato, badge por documento (FC, RU, CB, CM), cumplimiento general y autorizaciones |
| 3 | **Paginación** | Selector de tamaño de página y navegador con elipsis |
| 4 | **Modal detalle tienda** | Tabs entre tienda y sus autorizaciones; permite ver, subir, eliminar y validar/invalidar cada documento |
| 5 | **Modal visualización** | Renderiza el documento desde S3 vía `VisualizacionDocumentoComprobanteComponent` |
| 6 | **Modal subir documento** | Formulario para seleccionar tipo, nombre y archivo |
| 7 | **Modal nueva autorización** | Input para el nombre del delegado antes de abrir el modal de subir |

---

## **Flujo de inicialización**

```
ngOnInit()
  -> isPlatformBrowser() — solo ejecuta en navegador
  -> fetchData()
     -> Promise.all([
          tiendasService.consultarTienda(),
          consumoGenericoService.consultarGenerico('TiendaDocuments')
        ])
     -> decompressionService.decompressGzip() x2
     -> buildTiendaDocumentStatus(tiendas, documentos)
  -> tiendas = resultado
  -> tiendasFiltradas = [...tiendas]
  -> isLoading = false
```

---

## **Métodos clave**

### **`buildTiendaDocumentStatus()` *(privado)***

Cruza tiendas con sus documentos de MongoDB. Por cada tienda separa registros de tienda y de autorizaciones, aplana los documentos, toma el más reciente por tipo con `getUltimoDocumento()` (función interna), calcula `cumplimiento` (4 docs + todos validados → `'completo'`) y agrupa autorizaciones con el mismo proceso para 5 tipos.

### **`subirDocumento()`**

Valida campos → construye `s3Key` (`{IdTienda}_{Nombre}_{TipoDoc}` o con `AUTH{idAut}` si es autorización) → convierte a Base64 → sube a S3 → inserta registro en MongoDB → recarga datos y actualiza modales.

### **`eliminarDocumento(doc)`**

Pide confirmación → localiza el registro MongoDB por `sourceRecordId` → reconstruye el array `Documents` sin el elemento → actualiza en MongoDB → recarga.

### **`toggleValidado(doc)`**

Invierte `doc.Validado` → clona el array `Documents` → actualiza el elemento en `indexEnArray` → persiste en MongoDB → recarga.

### **`eliminarAutorizacion(aut)`**

Pide confirmación → localiza todos los registros de MongoDB donde `IdAutorizacion === aut.id` → llama a `eliminarGenerico` por cada uno → retorna tab a `'tienda'` → recarga.

---

## **Estados de la vista**

| Estado | Qué muestra |
| --- | --- |
| Cargando | Spinner (CoreUI `SpinnerModule`) |
| Con datos | Tabla paginada con badges de cumplimiento |
| Error | Solo `console.error` — sin feedback visual (deuda técnica) |
| Vacío | Tabla vacía por comportamiento natural de `*ngFor` |

---

## **Interfaces**

### **`TiendaDocumentStatus`**

| Campo | Tipo | Descripción |
| --- | --- | --- |
| `Id`, `NombreTienda` | `string` | Identificador y nombre de la tienda |
| `FC`, `RU`, `CB`, `CM` | `DocumentInfo | null` | Documento más reciente de cada tipo |
| `cumplimiento` | `'completo' | 'pendiente' | 'incompleto'` | Calculado: completo = 4 docs y todos validados |
| `autorizaciones` | `AutorizacionInfo[]` | Delegados con sus documentos |
| `sourceRecords` | `any[]` | Registros crudos de MongoDB — necesarios para operaciones CRUD sin re-consultar |

### **`DocumentInfo`**

| Campo | Tipo | Descripción |
| --- | --- | --- |
| `s3Key` | `string` | Ruta del archivo en S3 |
| `Validado` | `boolean` | Si un admin ya lo validó |
| `indexEnArray` | `number` | Posición en el array `Documents` de MongoDB |
| `sourceRecordId` | `string` | `_id` del registro MongoDB que contiene este doc |

### **`AutorizacionInfo`**

| Campo | Tipo | Descripción |
| --- | --- | --- |
| `id` | `string` | `IdAutorizacion` en MongoDB |
| `nombre` | `string` | Nombre del delegado |
| `FC`, `RU`, `CB`, `CM`, `CA` | `DocumentInfo | null` | 5 documentos requeridos por delegado |
| `cumplimiento` | `'completo' | 'pendiente' | 'incompleto'` | Completo = 4 docs (FC, RU, CB, CA) y todos validados |

---

## **Subcomponentes**

| Componente | Selector | Qué hace |
| --- | --- | --- |
| `VisualizacionDocumentoComprobanteComponent` | `app-visualizacion-documento-comprobante` | Renderiza un documento desde S3 dado su `s3Key` y una opción de visualización (`docOpcion = '2'`) |

---

## **Changelog**

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-03-16 | Iker Acevedo Vargas | Versión inicial |