

## Autor: Iker Acevedo
Fecha creacion: 2026-06-02

Estado: produccion

## Lambda: ApiLambdaSincronizacionUnion


**Accionador:** API Gateway

**AOT:** No

---

## ¿Qué hace?

Procesa lotes de pedidos por las transportadoras integradas (Interrapidísimo, Envia, XCargo, Servientrega, D2E). Por cada pedido crea el preenvío en la API de la transportadora, obtiene la guía en PDF, la modifica si en las variables de entorno esta activa a la auditoria, (Solo para Interrapidismo) y la guarda en S3. Finalmente actualiza el estado del pedido en MongoDB e inserta el registro en la colección de pedidos correspondiente.

---

## Accionador

| Método | Ruta | Auth |
| ------ | ---- | ---- |
| `POST` | API Gateway — `ApiLambdaCargarGuias` | Bearer token (Cognito) |

---

## Request

```json
{
  "IdCarga": "10000",
  "TipoEtiqueta": "Sticker",
  "CargarAlarmas": true
}
```

| Campo | Tipo | Requerido | Descripción |
| ----- | ---- | --------- | ----------- |
| `IdCarga` | `string` | Sí | ID del lote de carga en MongoDB (colección `CargaPedido`) |
| `TipoEtiqueta` | `string` | Sí | Formato de la guía: `"Sticker"` o `"Mediana"` |
| `CargarAlarmas` | `bool` | No | Si es `false` omite la validación de alarmas (duplicados, % devoluciones). Default: `true` |

El token de autenticación va en el header:
```
Token: eyJraWQi...
```

---

## Response

### Exitoso

```json
{
  "Resultado": "Carga {IdCarga} procesada exitosamente",
  "Error": false,
  "Mensaje": null
}
```

### Errores

| Código | Cuándo |
| ------ | ------ |
| `500` | Excepción no controlada en el handler principal |

Los errores por pedido individual no detienen el lote — el pedido queda con `Estado: "Rechazado"` y el campo `ERROR` con el mensaje en MongoDB.

---

## Flujo interno

```
FunctionHandler (Function.cs)
  -> Lee lote de MongoDB: ObtenerCargaPorIdAsync("CargaPedido")
  -> Por cada pedido según TRANSPORTADORA:
  
     [INTERRAPIDISIMO] CargaInterUseCase.procesarRegistroCarga
       -> procesarAlarmas (valida duplicados y % devoluciones)
       -> Construye AdmisionInter con datos del pedido
       -> API Inter: InsertarAdmision → obtiene numeroPreenvio
       -> API Inter: ObtenerEtiqueta (PDF en Base64)
       -> PdfUtils.ModifyPdf → modifica PDF si MODIFICACION_ETIQUETA=true
       -> S3: guarda PDF modificado en bucket-guias-inter-prod/{numeroPreenvio}.txt
       -> [Auditoría] InformacionEtiquetaInter.ObtenerDatosEtiqueta → MongoDB: AuditoriaEtiquetasInter
       -> MongoDB: actualiza CargaPedido (Estado=Cargado)
       -> MongoDB: inserta en PedidosInter
     
     [ENVIA] CargaEnviaUseCase.procesarRegistroCarga
     [D2 / XCargo] CargaXCargoUseCase.procesarRegistroCarga
     [SERVIENTREGA] CargaServientregaUseCase.procesarRegistroCarga
     [D2E] CargaD2EUseCase.procesarRegistroCarga

  -> Dispara ActualizaInventarioLotes en background (sin esperar)
  -> Envía SMS de notificación con resumen de carga
```

Los pedidos se procesan en lotes de 3 en paralelo (`Task.WhenAll` con batch de 3).

---

## Modificación de etiqueta Inter (PDFUtils)

Cuando `TipoEtiqueta = "Sticker"` y `MODIFICACION_ETIQUETA = true`, el PDF recibido de Inter se modifica antes de guardarse en S3:

1. **CubrirCaja** — dibuja un rectángulo blanco sobre un area dentro de la guia, el cuadrado es blanco con tamaño personalizable.

2. **AgregarDireccionAlFinal** — imprime la dirección original del pedido con la que se sube la guia en la zona inferior de la guía. Si la dirección supera el ancho disponible se parte automáticamente en múltiples líneas.

La fuente Arial esta embebida dentro de la compilacion de la lambda para que en produccion no falle porque no tiene el archivo .ttf

---

## Auditoría de etiquetas Inter (AuditoriaEtiquetasInter)

Cuando `AUDITORIA_ETIQUETA = true` **y** `TipoEtiqueta = "Sticker"`, después de obtener el PDF de Inter se extrae información de la guía usando iText7 y se guarda en MongoDB:

**Colección:** `AuditoriaEtiquetasInter`

| Campo | Origen | Descripción |
| ----- | ------ | ----------- |
| `NumeroPreenvio` | API Inter | Número de guía asignado por Inter |
| `NombreInterrapidisimo` | PDF extraído | Nombre del destinatario según Inter |
| `DireccionInterrapidisimo` | PDF extraído | Dirección según Inter |
| `CiudadDestinatario` | PDF extraído | Ciudad según Inter |
| `ValorCobrar` | PDF extraído | Valor a cobrar impreso en la guía |
| `Nombre` | MongoDB/Carga | Nombre tal como lo subió el usuario |
| `Direccion` | MongoDB/Carga | Dirección tal como la subió el usuario |

La auditoría está en un `try/catch` independiente — si falla, el proceso de carga continúa sin interrumpirse.

La auditoría **no corre** para `TipoEtiqueta = "Mediana"` porque el PDF de ese formato tiene una estructura diferente.

---

## Variables de entorno

| Variable | Descripción | Valores |
| -------- | ----------- | ------- |
| `MODIFICACION_ETIQUETA` | Activa o desactiva la modificación del PDF de Inter | `"true"` / `"false"` |
| `AUDITORIA_ETIQUETA` | Activa o desactiva la auditoría de etiquetas en MongoDB | `"true"` / `"false"` |
| `URL_SERVICIO_INTER` | URL base de la API de Interrapidísimo | URL |
| `URL_SERVICIO_AWS` | URL base de servicios internos AWS | URL |
| `CADENA_CONEXION` | Cadena de conexión MongoDB (encriptada AES) | String encriptado |
| `DATABASE_NAME` | Nombre de la base de datos MongoDB | `"LogighoDB"` |
| `ID_CLIENTE` | ID cliente crédito Inter (encriptado AES) | String encriptado |
| `USER_AUTH` | Usuario autenticación Inter (encriptado AES) | String encriptado |
| `TOKEN_AUTH` | Token autenticación Inter (encriptado AES) | String encriptado |
| `SUCURSAL_GENERICA` | Sucursal genérica Inter (encriptado AES) | String encriptado |

---

## Dependencias externas

| Servicio | Uso |
| -------- | --- |
| `API Interrapidísimo` | Crear preenvío (`InsertarAdmision`) y obtener PDF de guía (`ObtenerBase64PdfPreGuiaFormatoPeq`) |
| `S3 — bucket-guias-inter-prod` | Almacenar el PDF modificado de cada guía Inter como `{numeroPreenvio}.txt` en Base64 |
| `API Envia` | Crear guías Envia |
| `API XCargo / D2` | Crear guías XCargo y D2 |
| `API Servientrega` | Crear guías Servientrega |
| `Lambda ApiLambdaInventarioCarga` | Actualización de inventario por lotes al finalizar la carga (llamada en background) |

---

## Historial de cambios

| Fecha | Autor | Cambio |
| ----- | ----- | ------ |
| 2026-06-18 | Iker Acevedo | Modificación de etiqueta Sticker Inter: se oculta tabla NODO/ZONA/MANZANA con rectángulo blanco y se imprime la dirección original del pedido en la zona inferior del PDF. Fuente Arial embebida en el DLL para compatibilidad con Lambda/Linux. |
| 2026-06-18 | Iker Acevedo | Auditoría de etiquetas Inter: extracción de datos del PDF con iText7 y persistencia en colección `AuditoriaEtiquetasInter`. Solo activa para Sticker y cuando `AUDITORIA_ETIQUETA=true`. |

---

## Observaciones

- La auditoría falla silenciosamente — un error en `InformacionEtiquetaInter` no bloquea la creación de la guía.
