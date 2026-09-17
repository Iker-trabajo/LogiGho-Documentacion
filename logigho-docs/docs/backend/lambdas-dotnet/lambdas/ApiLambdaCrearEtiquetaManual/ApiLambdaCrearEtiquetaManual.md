

## Autor: Iker Acevedo
Fecha creacion: 2026-09-14

Estado: preprod

## Lambda: ApiLambdaCrearEtiquetaManual


**Accionador:** Step Function (`arn:aws:states:::lambda:invoke`) y API Gateway (reimpresión manual)

**AOT:** No

---

## ¿Qué hace?

Genera (o reimprime) el PDF de la etiqueta/rótulo de una guía ya creada, para las transportadoras integradas (Interrapidísimo, Envia, XCargo, Servientrega, D2E, TCC). Por cada número de guía, consulta la API de la transportadora correspondiente para obtener la URL del rótulo, descarga el PDF, lo re-aloja en el bucket servible general y actualiza `UrlGuia` en `PedidosInter`.

Tiene **dos llamadores con formas de evento distintas**:

1. **Step Function** (`CrearEtiquetaManualStateMachine`) — se dispara automáticamente después de que `ApiLambdaCrearGuia` o `ApiLambdaCargarGuias` despachan una guía con éxito, llamando a `/creacionEtiqueta`. El evento llega como el JSON de `EtiquetaRequest` directo, sin envoltorio de API Gateway.
2. **API Gateway** — para reimpresión manual desde el frontend. El body real viene envuelto dentro de `APIGatewayHttpApiV2ProxyRequest`, como string en la propiedad `"body"`.

El handler detecta cuál de los dos formatos llegó (`eventoJson.TryGetValue("body", ...)`) antes de parsear el `EtiquetaRequest`.

---

## Accionador

| Origen | Forma del evento | Uso |
| ------ | ----------------- | --- |
| Step Function (`arn:aws:states:::lambda:invoke`) | `EtiquetaRequest` directo como evento completo | Generación automática de etiqueta tras crear guía (`ApiLambdaCrearGuia`/`ApiLambdaCargarGuias` → `/creacionEtiqueta`) |
| API Gateway | `APIGatewayHttpApiV2ProxyRequest`, body como string | Reimpresión manual desde el frontend |

Reintentos y manejo de fallas de infraestructura (no de negocio) están configurados en la Step Function: `Retry` sobre excepciones `Lambda.*` y `Catch` → publica en un tópico SNS (`TopicoNotificacionesPreprod`) ante cualquier error no controlado (`States.ALL`).

---

## Request

```json
{
  "NumerosGuia": ["603974894"],
  "TipoEtiqueta": "Sticker",
  "Token": "eyJraWQi...",
  "Transportadora": "TCC"
}
```

| Campo | Tipo | Requerido | Descripción |
| ----- | ---- | --------- | ----------- |
| `NumerosGuia` | `string[]` | Sí | Números de guía a generar/reimprimir. Se procesan en lotes de 3 en paralelo |
| `TipoEtiqueta` | `string` | Sí | `"Sticker"` o `"Mediana"` |
| `Token` | `string` | Sí | Token Cognito, usado para subir el PDF al bucket servible |
| `Transportadora` | `string` | Sí | Si no coincide con ninguna transportadora soportada, cae por defecto a `"ENVIA"` |

---

## Response

### Exitoso

```json
{
  "Resultado": ["603974894"],
  "Error": false,
  "Mensaje": null
}
```

`Resultado` trae, por cada número de guía, el mismo número si tuvo éxito o `"Error {mensaje}"` si falló — **`GenerarEtiqueta` nunca lanza excepción**, siempre devuelve un string (a propósito: así un fallo de una guía puntual no bloquea el resto del lote ni dispara el `Catch` de la Step Function, que está reservado para fallas reales de infraestructura).

### Errores

| Código | Cuándo |
| ------ | ------ |
| Excepción no controlada | Falla en el parseo del evento o en la resolución de dependencias — sí propaga y activa el `Catch` de la Step Function |

---

## Flujo interno

```
FunctionHandler (Function.cs)
  -> Detecta forma del evento (Step Function vs API Gateway) y parsea EtiquetaRequest
  -> Por cada NumeroGuia, en lotes de 3 en paralelo, según Transportadora:

     [INTERRAPIDISIMO] CargaInterUseCase.GenerarEtiqueta
     [ENVIA]           CargaEnviaUseCase.GenerarEtiqueta
     [D2 / XCargo]     CargaXCargoUseCase.GenerarEtiqueta
     [SERVIENTREGA]    CargaServientregaUseCase.GenerarEtiqueta
     [D2E]             CargaD2EUseCase.GenerarEtiqueta

     [TCC] CargaTccUseCase.GenerarEtiqueta
       -> API TCC: remesas/impresionrotulosV2 (identificacion=NIT, remesas=[numeroGuia])
       -> Prioriza UrlRotulosDefinitivo; si TCC aún no lo tiene listo, usa UrlRotulosTemporal
       -> Descarga el PDF de esa URL (limpiando "amp;") y lo convierte a Base64
       -> S3: bucket-guias-tcc-preprod/{numeroGuia}.txt
       -> GuiaPdfHelper.SubirPdfConReintentos -> bucket servible general
       -> MongoDB: ActualizarUrlGuiaAsync("PedidosInter", numeroGuia, urlGuia)
       -> Si algo falla, retorna "Error {mensaje}" (nunca lanza)

  -> Responde con el arreglo de resultados (guía o error) por número de guía
```

---

## Transportadora TCC (CargaTccUseCase)

Integración con TCC (`ApiLambdaCrearEtiquetaManual.Aplicacion.CasosUso.Carga.CargaTccUseCase`). `OpcionConsumoTcc = 9` en el switch de `Generico.cs` de esta lambda.

A diferencia de `ApiLambdaCrearGuia`/`ApiLambdaCargarGuias`, esta lambda **no** despacha ni liquida — solo consulta el rótulo de una guía TCC que ya existe. Por eso las entidades TCC de esta lambda (`Dominio/Entidades/TCC/AdmisionTcc.cs`) son mucho más chicas: solo `ImprimirRotuloRequest`/`ImprimirRotuloResponse`, sin `AdmisionTcc`, `LiquidacionRequest` ni el resto de DTOs de despacho/liquidación que sí tienen las otras dos lambdas.

### `remesas/impresionrotulosV2`

```json
// Request
{ "identificacion": "{TCC_NIT desencriptado}", "remesas": ["603974894"] }
```

```json
// Response (campos relevantes)
{
  "UrlRotulosTemporal": "https://.../Informesdsp.aspx?...-TEMPORAL...",
  "UrlRotulosDefinitivo": "https://.../Informesdsp.aspx?...-DEFINITIVO...",
  "codigoresultado": "0",
  "mensajeresultado": "..."
}
```

Si `codigoresultado != "0"`, se lanza excepción con `mensajeresultado` — capturada por el `catch` de `GenerarEtiqueta`, que la convierte en `"Error {mensaje}"` en vez de propagarla.

### Prioridad de URL

```csharp
string urlRotuloTcc = !string.IsNullOrWhiteSpace(rotulo.UrlRotulosDefinitivo)
    ? rotulo.UrlRotulosDefinitivo
    : rotulo.UrlRotulosTemporal;
```

TCC puede tardar en generar el rótulo definitivo; mientras tanto expone uno temporal. Se prioriza el definitivo y se cae al temporal solo si el primero viene vacío.

### Rótulo PDF

Mismo patrón que en `ApiLambdaCargarGuias` — ver [ApiLambdaCargarGuias → Rótulo PDF](../ApiLambdaCargarGuias/ApiLambdaCargarGuias.md#rótulo-pdf) para el detalle de limpieza de `amp;`, descarga, subida a bucket propio y re-alojado con `GuiaPdfHelper` (incluyendo el fallback a la URL directa de TCC si el re-alojado falla).

### Headers de la API TCC

Mismos headers que en las otras dos lambdas TCC: `AccessToken`, `User-Agent: LogiGho-Lambda/1.0`, `Accept: application/json`.

---

## Variables de entorno

| Variable | Descripción | Valores |
| -------- | ----------- | ------- |
| `URL_SERVICIO_TCC` | URL base de la API de TCC | `https://testsomos.tcc.com.co/api/clientes/` (sandbox) |
| `TCC_ACCESS_TOKEN` | AccessToken de la API TCC (encriptado AES) | String encriptado |
| `TCC_NIT` | NIT/clave del cliente TCC (encriptado AES) | String encriptado |
| `CADENA_CONEXION` | Cadena de conexión MongoDB (encriptada AES) | String encriptado |
| `DATABASE_NAME` | Nombre de la base de datos MongoDB | `"LogighoDB"` |

TCC en esta lambda **no** usa `TCC_CUENTA_PAQUETERIA`/`TCC_CUENTA_MENSAJERIA` ni `TCC_PERMITIR_LIQUIDACION_FALLIDA` — no calcula flete ni hace liquidación, solo consulta el rótulo de una guía ya despachada.

---

## Dependencias externas

| Servicio | Uso |
| -------- | --- |
| `API Interrapidísimo` / `Envia` / `XCargo` / `Servientrega` / `D2E` | Obtener PDF de rótulo por transportadora |
| `API TCC` | `remesas/impresionrotulosV2` — obtener URL del rótulo de una remesa ya despachada |
| `S3 — bucket-guias-tcc-preprod` | Almacenar el PDF de cada guía TCC como `{numeroGuia}.txt` en Base64 |
| `Lambda ApiLambdaPutObjectAOT` | Re-alojar el PDF en el bucket servible general vía `GuiaPdfHelper` |
| `SNS — TopicoNotificacionesPreprod` | Notificación ante fallas de infraestructura no controladas (vía `Catch` de la Step Function) |

---

## Infraestructura — Step Function

`CrearEtiquetaManualStateMachine` (CloudFormation, `pedidos-infra-preprod.yaml`): un solo estado `Task` que invoca esta lambda vía `arn:aws:states:::lambda:invoke`, con `Retry` sobre excepciones `Lambda.*Exception` y `Catch` → `SNS Publish` sobre `States.ALL`. El ASL es agnóstico al payload (no tiene lógica específica de TCC ni de ninguna transportadora) — todo el ruteo por transportadora vive dentro de esta lambda.

`CrearEtiquetaManualStateMachineRole` (IAM): permite `lambda:InvokeFunction` sobre esta lambda y `sns:Publish` sobre el tópico de notificaciones.

---

## Historial de cambios

| Fecha | Autor | Cambio |
| ----- | ----- | ------ |
| 2026-09-01 | Iker Acevedo | Integración TCC (HU 3.3 — reimpresión de etiqueta): `CargaTccUseCase.GenerarEtiqueta`, entidades `ImprimirRotuloRequest`/`ImprimirRotuloResponse`, consumo de `remesas/impresionrotulosV2` con prioridad `UrlRotulosDefinitivo` > `UrlRotulosTemporal`. |
| 2026-09-03 | Iker Acevedo | Fix wiring: el caso `["TCC"]` en el diccionario de transportadoras de `Function.cs` llamaba por error a `_cargaD2eUseCase.GenerarEtiqueta` en vez de `_cargaTccUseCase.GenerarEtiqueta`. |

---

## Observaciones

- `GenerarEtiqueta` **nunca lanza excepción** para ninguna transportadora (incluida TCC) — siempre retorna un string, sea el número de guía o `"Error {mensaje}"`. Es intencional: el `Catch` de la Step Function está reservado para fallas reales de infraestructura (Lambda caída, timeout, etc.), no para errores de negocio por guía individual.
- Esta lambda no valida cobertura de ciudad ni calcula flete para TCC — asume que la guía ya fue despachada exitosamente por `ApiLambdaCrearGuia` o `ApiLambdaCargarGuias`, que sí hacen esas validaciones antes de llegar acá.
- El fallback de `UrlRotulosDefinitivo` a `UrlRotulosTemporal` puede devolver una URL cuyo PDF cambie más adelante (cuando TCC termine de generar el definitivo) — si se reimprime la misma guía más tarde, es posible obtener el rótulo definitivo en un segundo intento.
