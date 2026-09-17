

## Autor: Iker Acevedo
Fecha creacion: 2026-06-02

Estado: produccion

## Lambda: ApiLambdaCargarGuias


**Accionador:** API Gateway

**AOT:** No

---

## ¿Qué hace?

Es la lambda princiapl que procesa lotes de pedidos por las transportadoras integradas (Interrapidísimo, Envia, XCargo, Servientrega, D2E, TCC). Por cada pedido se acciona el caso de uso de acuerdo a la transportadora, crea el preenvío en la API de la transportadora, obtiene la guía en PDF y la guarda en el bucket S3 de cada transportadora. Finalmente actualiza el estado del pedido en MongoDB e inserta el registro en la colección de pedidos de `PedidosInter`.

---

## Accionador

| Método | Ruta | Autenticacion |
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
| `CargarAlarmas` | `bool` | No | Si es `false` omite la validación de alarmas (duplicados, % devoluciones). Por defecto esta activo |

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

     [TCC] CargaTccUseCase.procesarRegistroCarga
       -> procesarAlarmas (valida duplicados y % devoluciones)
       -> Valida cobertura: busca CIUDAD ORIGEN y CIUDAD (código DANE8) en CiudadesTCC con Estado=ACTIVA
       -> Define paquetería/mensajería según PESO (>5kg = paquetería, cuenta/tipo de servicio distintos)
       -> API TCC: tarifas/v6/consultarliquidacion (OBLIGATORIA, secuencial y antes del despacho)
          - Si falla y no es el caso conocido de sandbox (ver abajo) -> rechaza el pedido sin crear nada en TCC
       -> API TCC: remesas/grabardespacho7 -> obtiene numeroremesa y urlrotulos
       -> Descarga el PDF de urlrotulos (limpiando "amp;") y lo re-aloja:
          - S3: bucket-guias-tcc-preprod/{remesa}.txt
          - GuiaPdfHelper.SubirPdfConReintentos -> bucket servible general (fallback a la URL de TCC si falla)
       -> MongoDB: actualiza CargaPedido (Estado=Cargado)
       -> MongoDB: inserta en PedidosInter (Trayecto = tipoenvio de la liquidación, ej. "ZONAL")

  -> Dispara ActualizaInventarioLotes en background (sin esperar)
  -> Envía SMS de notificación con resumen de carga
```

Los pedidos se procesan en lotes de 3 en paralelo (`Task.WhenAll` con batch de 3).

---

## Modificación de etiqueta Inter (PDFUtils)

Por reglas de administracion, se inicio un proceso de personalizado con las guias de interrapidisimo. Para esto se configuro de tal forma que cuando `TipoEtiqueta = "Sticker"` y `MODIFICACION_ETIQUETA = true`, el PDF recibido de Inter se modifica antes de guardarse en S3:

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

## Transportadora TCC (CargaTccUseCase)

Integración con TCC (`ApiLambdaCargarGuias.Aplicacion.CasosUso.Carga.CargaTccUseCase`). Endpoint base: `URL_SERVICIO_TCC` (`https://testsomos.tcc.com.co/api/clientes/` en sandbox). `OpcionConsumoTcc = 9` en el switch de `Generico.cs`.

### Paquetería vs Mensajería

TCC maneja dos cuentas y tipos de servicio distintos según el peso del pedido:

| Peso (`PESO`) | Unidad de negocio | Cuenta | Tipo de servicio | Clase de empaque |
| -------------- | ------------------ | ------ | ------------------ | ------------------ |
| `> 5 kg` | `1` (paquetería) | `TCC_CUENTA_PAQUETERIA` | `TISE_NORMAL_PAQ` | `CLEM_CAJA` |
| `<= 5 kg` | `2` (mensajería) | `TCC_CUENTA_MENSAJERIA` | `TISE_NORMAL_MEN` | `CLEM_PEQUENA` |

### Cobertura por ciudad

`CIUDAD ORIGEN` y `CIUDAD` del pedido (código DANE8) se buscan en la colección `CiudadesTCC`, filtrando `Estado == "ACTIVA"` (la colección trae registros `INACTIVA` del cargue inicial del Excel de TCC — sin el filtro se aceptaban ciudades sin cobertura real). Si alguna de las dos no aparece activa, el pedido se rechaza antes de tocar la API de TCC:

```
TCC no tiene cobertura para la ciudad origen '{codigo}' o destino '{codigo}'.
```

### Liquidación obligatoria (`tarifas/v6/consultarliquidacion`)

Antes de crear el despacho, siempre se consulta la tarifa. Es una regla de negocio dura: **si la liquidación falla, el pedido se rechaza sin crear nada en TCC** — evita guías huérfanas con flete/sobreflete en 0 por defecto.

- `totalFlete` = concepto `idconcepto == 1` ("FLETE") de la respuesta.
- `totalSobreflete` = suma de los demás conceptos, **agrupados por `idconcepto` tomando el valor máximo de cada grupo** (TCC duplica conceptos como `58 - FLETE MANEJO` dos veces con el mismo valor; sumar sin agrupar duplicaba el sobreflete).
- `Trayecto` en `PedidosInter` se llena con `total.tipoenvio` de la respuesta de liquidación (ej. `"ZONAL"`, `"URBANO"`) — TCC ya clasifica el trayecto según las ciudades, se reutiliza en vez de recalcularlo.

#### Bypass de sandbox para recaudo (`TCC_PERMITIR_LIQUIDACION_FALLIDA`)

El sandbox de TCC no soporta recaudo (confirmado por TCC: solo funciona en producción), y la liquidación siempre responde `codigoResultado: "-2"` con el mensaje `"No se encontró configuración de recaudo para el cliente consultado."` para pedidos con contra pago. Sin un mecanismo de excepción, ningún pedido de prueba con recaudo podría probarse en preprod.

Variable de entorno `TCC_PERMITIR_LIQUIDACION_FALLIDA` (`"true"`/`"false"`, **ausente = `false`**, solo activa en `aws-lambda-tools-pruebas.json`, nunca en producción):

- Si `codigoResultado != "0"` **y** el mensaje contiene `"recaudo"` **y** el bypass está activo → el pedido continúa con `Total Flete`/`Total Sobreflete` en `0` y se marca:
  ```
  item["ObservacionesFleteTCC"] = "Liquidacion con recaudo no soportada en sandbox TCC — flete/sobreflete no confiables (solo pruebas)"
  ```
- Cualquier otro fallo de liquidación (o el mismo fallo con el bypass desactivado) rechaza el pedido normalmente.

### Despacho (`remesas/grabardespacho7`)

- `respuesta != "0"` → rechaza el pedido con el mensaje de TCC.
- `remesa` se intenta parsear como `long` para `NumeroPreenvio` (debe ser entero, no string, para ser consistente con el resto de transportadoras); si no parsea, se guarda el string sin ceros a la izquierda como respaldo.

### Rótulo PDF

`grabardespacho7` devuelve `urlrotulos` con caracteres `amp;` (según la documentación de TCC hay que quitarlos para que la URL sea válida). El flujo:

1. Limpia `urlrotulos` (`.Replace("amp;", "")`).
2. Descarga el PDF de esa URL y lo convierte a Base64 (`DescargarPdfComoBase64Async`).
3. Sube el Base64 a `bucket-guias-tcc-preprod/{remesa}.txt` (bucket propio, mismo patrón que las otras transportadoras).
4. Re-aloja con `GuiaPdfHelper.SubirPdfConReintentos` hacia el bucket servible general (requiere que `"tcc"` esté en la lista `transportadorasValidas` de `ApiLambdaPutObjectAOT`).
5. Si la descarga del PDF falla, o si el re-alojado devuelve vacío (`GuiaPdfHelper` nunca lanza excepción, solo retorna `""` ante fallo), `UrlGuia` cae de vuelta a la URL directa de TCC.

### Headers de la API TCC

```
AccessToken: {TCC_ACCESS_TOKEN desencriptado}
User-Agent: LogiGho-Lambda/1.0
Accept: application/json
```

`Clave` en el body del despacho y `AccessToken` en el header usan el mismo secreto — TCC no diferencia credencial de body vs credencial de header.

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
| `URL_SERVICIO_TCC` | URL base de la API de TCC | `https://testsomos.tcc.com.co/api/clientes/` (sandbox) |
| `TCC_ACCESS_TOKEN` | AccessToken de la API TCC (encriptado AES) | String encriptado |
| `TCC_NIT` | NIT/clave del cliente TCC (encriptado AES) — mismo valor para `clave` del body y `AccessToken` del header | String encriptado |
| `TCC_CUENTA_PAQUETERIA` | Cuenta TCC para envíos > 5kg | `"1485100"` |
| `TCC_CUENTA_MENSAJERIA` | Cuenta TCC para envíos <= 5kg | `"5625200"` |
| `TCC_PERMITIR_LIQUIDACION_FALLIDA` | Bypass del fallo de recaudo en sandbox (ver sección TCC arriba). **Solo en preprod, nunca en producción** | `"true"` / ausente (= `false`) |

---

## Dependencias externas

| Servicio | Uso |
| -------- | --- |
| `API Interrapidísimo` | Crear preenvío (`InsertarAdmision`) y obtener PDF de guía (`ObtenerBase64PdfPreGuiaFormatoPeq`) |
| `S3 — bucket-guias-inter-prod` | Almacenar el PDF modificado de cada guía Inter como `{numeroPreenvio}.txt` en Base64 |
| `API Envia` | Crear guías Envia |
| `API XCargo / D2` | Crear guías XCargo y D2 |
| `API Servientrega` | Crear guías Servientrega |
| `API TCC` | Consultar liquidación (`tarifas/v6/consultarliquidacion`) y crear despacho (`remesas/grabardespacho7`) |
| `S3 — bucket-guias-tcc-preprod` | Almacenar el PDF de cada guía TCC como `{remesa}.txt` en Base64 |
| `Lambda ApiLambdaPutObjectAOT` | Re-alojar el PDF de TCC (y de las demás transportadoras) en el bucket servible general vía `GuiaPdfHelper` |
| `Lambda ApiLambdaInventarioCarga` | Actualización de inventario por lotes al finalizar la carga (llamada en background) |

---

## Historial de cambios

| Fecha | Autor | Cambio |
| ----- | ----- | ------ |
| 2026-06-18 | Iker Acevedo | Modificación de etiqueta Sticker Inter: se oculta tabla NODO/ZONA/MANZANA con rectángulo blanco y se imprime la dirección original del pedido en la zona inferior del PDF. Fuente Arial embebida en el DLL para compatibilidad con Lambda/Linux. |
| 2026-06-18 | Iker Acevedo | Auditoría de etiquetas Inter: extracción de datos del PDF con iText7 y persistencia en colección `AuditoriaEtiquetasInter`. Solo activa para Sticker y cuando `AUDITORIA_ETIQUETA=true`. |
| 2026-09-01 | Iker Acevedo | Integración TCC (HU 3.2 — carga masiva): `CargaTccUseCase`, entidades en `Dominio/Entidades/TCC`, liquidación previa obligatoria con fail-fast, mapeo paquetería/mensajería por peso, validación de cobertura por ciudad, re-alojado del PDF del rótulo en bucket propio y bucket servible. |
| 2026-09-03 | Iker Acevedo | Fix `NumeroPreenvio` de TCC: se guarda como `long`, no `string`, para ser consistente con el resto de transportadoras (ej. Envia usa `$numberLong` en Mongo). |
| 2026-09-03 | Iker Acevedo | Fix suma de sobreflete TCC: la respuesta de liquidación duplica conceptos con el mismo `idconcepto` (ej. `58 - FLETE MANEJO` dos veces) — se agrupa por `idconcepto` y se toma el valor máximo de cada grupo en vez de sumar todo. |
| 2026-09-03 | Iker Acevedo | `TCC_PERMITIR_LIQUIDACION_FALLIDA`: bypass controlado por variable de entorno para probar en preprod pedidos con recaudo, dado que el sandbox de TCC no soporta recaudo (confirmado por TCC). Ausente/`false` por defecto — nunca se activa en producción. |
| 2026-09-14 | Iker Acevedo | `Trayecto` en `PedidosInter` para TCC se llena con `total.tipoenvio` de la respuesta de liquidación (antes quedaba siempre vacío), reutilizando la clasificación que TCC ya calcula en vez de recalcularla. |

---

## Observaciones

- La auditoría falla silenciosamente — un error en `InformacionEtiquetaInter` no bloquea la creación de la guía.
- Para TCC, la liquidación es la única transportadora del set con una llamada previa obligatoria y fail-fast antes del despacho — el resto de transportadoras despachan directo y obtienen flete/sobreflete de la misma respuesta de creación de guía.
- `TCC_PERMITIR_LIQUIDACION_FALLIDA` es una válvula de escape exclusiva de pruebas: cuando se active en producción con credenciales reales (con recaudo funcionando), la variable debe quedar en `"false"` o ausente — de lo contrario un fallo real de liquidación pasaría con flete/sobreflete en `0`.
