---
autor: Iker Acevedo
fecha_creacion: 2026-10-25
ultima_actualizacion: 2026-10-25
estado: vigente
---

# Probar la API de guías en PreProd, a mano

Versión sin scripts de [prueba-preprod.md](prueba-preprod.md): cada paso es un comando o un clic. Lo hace Iker; la IA no escribe en AWS ni en la base. Todo lo que se necesita está en `C:\Users\Iker\Desktop\guias-preprod\`:

| Qué | Dónde |
| --- | --- |
| Los 4 zips con el código | `zips\` (`ApiLambdaGuias`, `ApiLambdaGuiasWorker`, `ApiLambdaGuiasCalculadora`, `ApiLambdaGuiasMcp`) |
| Cuerpos de las pruebas (JSON) | `pruebas\` |
| Colección y entorno de Postman | `guias-preprod.postman_collection.json` y `guias-preprod.postman_environment.json` |

> **Aviso.** Una guía válida llama a la Interrapidísimo **real** con la URL que tiene `ApiLambdaCargarGuias` en PreProd. Es una guía de verdad con datos ficticios: hay que anularla al terminar (§9).

---

## 1. ¿Hay que configurar algo en Azure DevOps?

**No, para esta prueba.** El despliegue de la API de guías no pasa por Azure DevOps todavía:

| Tema | Estado |
| --- | --- |
| Pipeline de CI (`pipelines/servicio-publico/ci.yml`) | Existe en el repo y solo **valida** (compila, prueba, empaqueta). Se dispara por la política de rama de `main` y `release/*` cuando haya PR; no se necesita para probar. Usa el grupo de variables `slack-notificaciones`, que ya usan los otros pipelines |
| Pipeline de CD de servicio-publico | **No existe.** El código se publica a mano (§4). Cuando se cree: un grupo de variables (Library) con las 6 variables cifradas con AES (`CADENA_CONEXION`, `ID_CLIENTE`, `USER_AUTH`, `TOKEN_AUTH`, `SUCURSAL_GENERICA`, `URL_SERVICIO_INTER`) y una conexión de servicio a AWS PreProd |
| Producción | Se hace por la consola web de AWS (decisión de Iker) |

---

## 2. Orden

1. Terraform (§3) → 2. Código y variables de las Lambdas (§4) → 3. Datos en Atlas (§5) → 4. Pruebas (§6 y §7) → 5. Pruebas de reglas con datos distintos (§8) → 6. Limpieza (§9).

---

## 3. Terraform

En PowerShell, con el perfil de PreProd:

```powershell
$env:AWS_PROFILE = "logigho-preprod-admin"; $env:AWS_REGION = "us-east-1"
aws sts get-caller-identity --query Account --output text
```

Debe imprimir `376313750428`. Si imprime otra cuenta, **parar**. Después:

```powershell
cd "C:\Users\Iker\Desktop\Logigho-repos\Terraform_Infraestructura_Logigho-guias\environments\preprod\guias"
terraform init
terraform plan -out guias-preprod5.tfplan
```

El plan debe mostrar **solo altas**: la función `ApiLambdaGuiasCalculadoraPreProd` (con su rol, su grupo de registros y su alarma de errores y de throttles), una política de invocación en cada uno de los roles de la API, el worker y el reconciliador, y las alarmas `documento-ausente` y `fallida-por-reintentos`. Si el plan quiere **destruir o reemplazar** una función existente, no aplicar y avisar.

```powershell
terraform apply guias-preprod5.tfplan
```

---

## 4. Código y variables de las Lambdas (consola web de AWS)

Región `us-east-1`, cuenta PreProd. Los **handlers, la memoria y el timeout ya los puso Terraform**; no se tocan.

### 4.1 Subir el código (una vez por función)

Lambda → la función → pestaña **Código** → **Cargar desde** → **Archivo .zip** → elegir el zip → **Guardar**.

| Función | Zip |
| --- | --- |
| `ApiLambdaGuiasCalculadoraPreProd` | `ApiLambdaGuiasCalculadora.zip` |
| `ApiLambdaGuiasPreProd` | `ApiLambdaGuias.zip` |
| `ApiLambdaGuiasWorkerPreProd` | `ApiLambdaGuiasWorker.zip` |
| `ApiLambdaGuiasReconciliadorPreProd` | `ApiLambdaGuiasWorker.zip` (el mismo; cambia el handler) |
| `ApiLambdaGuiasMcpPreProd` | no se toca (MCP pausado) |

Los zips pesan unos 18 a 19 MB: la consola los acepta directo (límite de 50 MB).

### 4.2 Variables de entorno (una vez por función)

Lambda → la función → **Configuración** → **Variables de entorno** → **Editar**. Los valores cifrados con AES se copian **tal cual** de `ApiLambdaCargarGuias` (PreProd) → Configuración → Variables de entorno (son el texto cifrado, no la clave en claro).

| Variable | Valor | Calculadora | API | Worker y Reconciliador |
| --- | --- | :-: | :-: | :-: |
| `CADENA_CONEXION` | copiar de `ApiLambdaCargarGuias` | ✔ | ✔ | ✔ |
| `ID_CLIENTE`, `USER_AUTH`, `TOKEN_AUTH`, `SUCURSAL_GENERICA`, `URL_SERVICIO_INTER` | copiar de `ApiLambdaCargarGuias` | | | ✔ |
| `ENTORNO` | `preprod` | ya está | agregar | agregar |
| `FUNCION_CALCULADORA_CUPO` | `ApiLambdaGuiasCalculadoraPreProd` | | agregar | agregar |
| `ESPERA_RESPUESTA_SEGUNDOS` | `20` | | agregar | |
| `OCULTAR_REMITENTE_ETIQUETA` | `true` | | | agregar |
| `TEXTOS_OCULTAR_ETIQUETA` | `LOGIGHO S.A.S` | | | agregar |
| `URL_SERVICIO_AWS` | **borrarla** (la reemplaza `URL_AVISO_INVENTARIO`) | | | borrar |
| `URL_AVISO_INVENTARIO` | **no crearla** en PreProd: sin ella el aviso a inventario se omite con un aviso en el registro y la métrica `InventarioOmitido` | | | — |

Las demás (`DATABASE_NAME`, `COLA_SOLICITUDES_URL`, `BUCKET_*`, `URL_CLOUDFRONT`, `TIMEOUT_INTER_SEGUNDOS`) ya las puso Terraform. `ApiLambdaGuiasCalculadoraPreProd` ya trae `DATABASE_NAME` y `ENTORNO`.

> Atención: `ApiLambdaCargarGuias` apunta a la **Interrapidísimo real**. Copiar `URL_SERVICIO_INTER` a las funciones nuevas hace que sus guías sean reales.

### 4.3 Comprobar

Lambda → **Probar** con un evento vacío `{}` sobre `ApiLambdaGuiasCalculadoraPreProd`: debería responder un JSON con `error.codigo = "SOLICITUD_INVALIDA"` (rechaza el evento vacío) y **no** una excepción. Eso demuestra que arrancó y que lee `CADENA_CONEXION`. Si falla con «CADENA_CONEXION no está configurada», falta la variable.

---

## 5. Datos en Atlas PreProd

Con `mongosh` (o la pestaña de comandos de Compass) conectado a la base `LogighoDB`. Las tres primeras son las **obligatorias**:

### 5.1 Cliente de prueba con la tienda `156783`

```javascript
db.ApiClientes.replaceOne(
  { ClienteId: "54s6t64cb84b8np5p5cc63sfjj" },
  {
    ClienteId: "54s6t64cb84b8np5p5cc63sfjj",
    Nombre: "Integrador de prueba (PreProd)",
    Nit: "",
    Estado: "ACTIVO",
    Tiendas: [
      { IdTienda: 156783, Nombre: "Tienda de prueba 156783", Transportadoras: ["INTERRAPIDISIMO"], Productos: [] }
    ],
    IpsPermitidas: [],
    TopeLote: 10
  },
  { upsert: true }
)
```

### 5.2 Cupo de la tienda (prepago, en `OBSERVAR`, sin límite diario)

```javascript
db.ApiCupoTiendas.insertOne({
  _id: 156783,
  Tienda: "Tienda de prueba 156783",
  Modo: "PREPAGO",
  LimiteCredito: 0,
  Aplicacion: "OBSERVAR",
  Suspension: { Activa: false },
  Parametros: { MargenEstimacionPct: 25, TopeLote: 50 },
  Saldo: { Confiable: 1000000, CalculadoEn: new Date() },
  ExposicionExterna: 0,
  Reservado: 0,
  Base: 1000000,
  Disponible: 1000000,
  Version: 1
})
```

### 5.3 Activar el producto de la tienda (hoy está `Inactivo`)

```javascript
db.Productos.updateOne({ idproducto: 1371 }, { $set: { estado: "Activo" } })
```

### 5.4 Comprobar lo que ya existe

```javascript
db.CostosTransporteTienda.countDocuments({ IdTienda: "156783" })   // filas de la tienda (entrega y devolución)
db.Ciudades.findOne({ "dane.mun8": "05001000", "providers.INTER": { $exists: true } }, { city: 1 })   // Medellín tiene cobertura
db.ApiSolicitudesGuia.countDocuments({})   // 0 antes de empezar
```

Los índices de las colecciones `Api*` los crea el código la primera vez que se usan.

---

## 6. Pruebas con Postman

1. **Import** → los dos archivos `guias-preprod.postman_*.json` de la carpeta.
2. Elegir el entorno **Guías PreProd** (arriba a la derecha) y llenar dos variables de tipo secreto: `clientSecret` (el contenido de `%USERPROFILE%\integrador-prueba-preprod-secreto.txt`) y `apiKey` (el de `%USERPROFILE%\mcp-api-key-preprod.txt`).
3. La colección obtiene y renueva el token sola. Ejecutar **carpeta por carpeta** (clic derecho → **Run**) o petición por petición, en este orden. Cada petición ya trae su prueba de código HTTP.

| Carpeta y petición | Esperado | Qué comprueba |
| --- | --- | --- |
| 0 · Salud | `200` | La API responde |
| 0 · Tiendas sin token | `401` | Autenticación |
| 0 · Tiendas sin API key | `403` | El gateway exige `x-api-key` |
| 1 · Tiendas del cliente | `200`, aparece `156783` | Perfil del cliente en Atlas |
| 1 · Transportadoras | `200`, `INTERRAPIDISIMO` | |
| 2 · **Crear guía válida** | `201` con `numeroGuia` si el worker termina en 20 s; si no, `202` | Flujo completo: validación → cupo (función de cálculo) → cola → Inter |
| 2 · Consultar la solicitud | `200`; repetir hasta `estado: CREADA` y `numeroGuia` | `GET /v1/solicitudes/{id}` |
| 2 · Reintento misma clave y body | `200`, misma `solicitudId` | Idempotencia: no se crea otra guía en Inter |
| 2 · Misma clave, otro body | `422 IDEMPOTENCY_CONFLICTO` | |
| 2 · Sin `Idempotency-Key` | `400` | |
| 2 · Peso 0 | `400 VALIDACION` | Validación campo por campo |
| 2 · Valor declarado 1 000 | `400` | Mínimo de 45 000 |
| 2 · `totalRecaudo` > 0 | `400 CONTRAENTREGA_NO_DISPONIBLE` | Contraentrega fuera de v1 |
| 2 · Campo que no existe | `400 CAMPO_NO_PERMITIDO` | Contrato estricto |
| 2 · Ciudad sin cobertura | `422 CIUDAD_SIN_COBERTURA` | |
| 2 · Tienda ajena | `403 TIENDA_NO_PERMITIDA` | Permisos del cliente |
| 2 · Sin productos | `422 PRODUCTOS_REQUERIDOS` | Tienda Dropshipping |
| 2 · Solicitud inexistente | `404` | |
| 2 · Guía con asesor | `201`/`202` | Después, `CargaPedido.ASESOR = "Carlos Prueba"` |
| 3 · **Lote parcial** | `202`: 2 `ACEPTADA` y 1 `RECHAZADA` con `rechazoId` | Lote parcial |
| 3 · Consultar el lote | `200`; repetir hasta `progreso.terminado = true` | Estado por guía |
| 3 · Reenviar el mismo lote | `202`, mismas `solicitudId` | Idempotencia por guía |
| 3 · Claves repetidas en el lote | `202`; la segunda `RECHAZADA` `IDEMPOTENCY_REPETIDA_EN_LOTE` | |
| 3 · Lote de 11 (tope 10) | `422 LOTE_DEMASIADO_GRANDE` | Tope por cliente |
| 3 · Lote sin guías | `400` | |
| 3 · Lote inexistente | `404` | |
| 4 · `/productos`, `/cupo` | `403`/`404`/`500` | **Aún no existen** (fase 2) |

> Para repetir el lote parcial con guías nuevas cambia las tres `idempotencyKey` de `pruebas\lote-parcial.json` y pega el archivo en el cuerpo de la petición.

### Alternativa con `curl.exe` (PowerShell)

```powershell
$secreto = (Get-Content "$env:USERPROFILE\integrador-prueba-preprod-secreto.txt" -Raw).Trim()
$apiKey  = (Get-Content "$env:USERPROFILE\mcp-api-key-preprod.txt" -Raw).Trim()
$base    = "https://x2ijau5ine.execute-api.us-east-1.amazonaws.com/PreProd/v1"

# 1) token (se guarda en $token; no se imprime)
$token = (curl.exe -s -X POST "https://logigho-guias-integradores-preprod.auth.us-east-1.amazoncognito.com/oauth2/token" `
  -u "54s6t64cb84b8np5p5cc63sfjj:$secreto" -H "Content-Type: application/x-www-form-urlencoded" `
  -d "grant_type=client_credentials&scope=logigho/guias:crear logigho/guias:leer" | ConvertFrom-Json).access_token

# 2) salud y tiendas
curl.exe -s "$base/salud"
curl.exe -s "$base/tiendas" -H "Authorization: Bearer $token" -H "x-api-key: $apiKey"

# 3) crear una guía (cambiar la clave en cada guía nueva)
curl.exe -s -i -X POST "$base/guias" -H "Authorization: Bearer $token" -H "x-api-key: $apiKey" `
  -H "Idempotency-Key: curl-001" -H "Content-Type: application/json" --data-binary "@C:\Users\Iker\Desktop\guias-preprod\pruebas\guia-valida.json"

# 4) consultar (poner la solicitudId de la respuesta)
curl.exe -s "$base/solicitudes/<solicitudId>" -H "Authorization: Bearer $token" -H "x-api-key: $apiKey"

# 5) lote
curl.exe -s -i -X POST "$base/lotes" -H "Authorization: Bearer $token" -H "x-api-key: $apiKey" `
  -H "Content-Type: application/json" --data-binary "@C:\Users\Iker\Desktop\guias-preprod\pruebas\lote-parcial.json"
curl.exe -s "$base/lotes/<loteId>" -H "Authorization: Bearer $token" -H "x-api-key: $apiKey"
```

---

## 7. Qué verificar después de la guía válida y del lote

**Atlas (`LogighoDB`):**

```javascript
db.ApiSolicitudesGuia.find({}, { Estado: 1, NumeroGuia: 1, "Reserva.Monto": 1, "Reserva.Origen": 1, PasosCompletados: 1, IntentosCreacion: 1, LoteId: 1, MotivoFallo: 1, MotivoRevisionManual: 1 })
db.ApiMovimientosCupo.find().sort({ Fecha: -1 })          // una Reserva y, si el valor real difiere, un AjusteReserva
db.ApiCupoTiendas.findOne({ _id: 156783 }, { Reservado: 1, Disponible: 1 })
db.CargaPedido.find({ Origen: "API" }, { Estado: 1, IdCarga: 1, ASESOR: 1, LoteApi: 1, ERROR: 1, NumeroPreenvio: 1 })   // IdCarga "-1"; las del lote con LoteApi
db.PedidosInter.find({ Origen: "API" }, { Estado: 1, Numeropreenvio: 1, UrlGuia: 1, LoteApi: 1, Asesor: 1 })
db.ApiLotesGuia.find()                                    // los ítems con RechazoId y errores
```

Esperado: solicitud `Creada` con `Reserva.Origen = "Real"`; `Reservado` igual a la suma de las reservas vivas; en `CargaPedido` y `PedidosInter` `LoteApi` solo en las guías del lote. Abrir `UrlGuia`: la etiqueta **sin** «LOGIGHO S.A.S» en el remitente.

**CloudWatch (Registros y métricas):**

- Registros de `/aws/lambda/ApiLambdaGuiasPreProd`, `…WorkerPreProd`, `…CalculadoraPreProd`: buscar por `solicitudId`. Al arrancar, cada función escribe un resumen de la conexión (entorno, tipo, huella; sin credenciales).
- Métricas, espacio **`ApiPublicaGuias`**: `InventarioOmitido` (1 por guía creada, esperado en PreProd), y cuando aplique `DocumentoAusente`, `SolicitudFallidaPorReintentos`, `GuiaEnEsperaClienteBloqueado`, `EtiquetaSinTransformar`.
- SQS: cola `ApiGuias-Solicitudes-preprod` sin mensajes visibles y la DLQ (`…-DLQ-preprod`) en 0.

---

## 8. Pruebas de reglas cambiando datos

Cada una se prepara en Atlas, se prueba y se **deja como estaba**. El caso de «documento ausente» no se puede provocar a mano con fiabilidad (la fila de `CargaPedido` la inserta el propio worker segundos antes de usarla); está cubierto por las pruebas automáticas contra Mongo.

| Prueba | Preparar | Hacer | Esperado |
| --- | --- | --- | --- |
| **Sin cupo** | `db.ApiCupoTiendas.updateOne({_id:156783},{$set:{Aplicacion:"APLICAR",Disponible:0,Base:0,"Saldo.CalculadoEn":new Date()}})` | Crear guía válida (clave nueva) | `422 CUPO_INSUFICIENTE`; en `ApiSolicitudesGuia` queda `Fallida`; en `CargaPedido` queda `Rechazado` con `ERROR` «Sin cupo disponible para la cuenta…». Luego devolver el cupo y **repetir la misma clave**: crea una guía nueva (Fallida libera la clave) |
| **Límite diario** | `db.ApiCupoTiendas.updateOne({_id:156783},{$set:{"Parametros.GuiasPorDiaTienda":1}})` | Dos guías válidas seguidas con claves distintas | La segunda: `429 LIMITE_DIARIO_ALCANZADO`. Volver a dejarlo sin el campo: `$unset:{"Parametros.GuiasPorDiaTienda":""}` |
| **Cliente bloqueado** | `db.ApiClientes.updateOne({ClienteId:"54s6t64cb84b8np5p5cc63sfjj"},{$set:{Estado:"BLOQUEADO"}})` | Cualquier petición | `403 CLIENTE_BLOQUEADO` (la caché es de 30 s: esperar medio minuto). Una guía ya aceptada y encolada espera sin crearse; al volver a `ACTIVO` se crea |
| **Cinco intentos** | En el worker, cambiar `URL_SERVICIO_INTER` a `http://127.0.0.1:1` | Crear una guía válida | El worker la reintenta (intentos 1 a 4) y al quinto queda `Fallida` con `TRANSPORTADORA_NO_DISPONIBLE`, libera el cupo y salta la alarma `fallida-por-reintentos`. **Restaurar el valor original** |
| **Tope de lote** | `TopeLote` del cliente en 10 (ya está) | Lote de 11 | `422 LOTE_DEMASIADO_GRANDE` |

---

## 9. Limpieza

1. **Anular en Inter** las guías de prueba (por ahora la anulación de la API no existe: se hace por el medio habitual de LogiGho con el número de guía).
2. Devolver el cupo a `OBSERVAR` y `Disponible` 1 000 000; quitar `GuiasPorDiaTienda`; cliente en `ACTIVO`; restaurar `URL_SERVICIO_INTER`.
3. Si no se sigue probando: cliente en `BLOQUEADO`.

---

## 10. Si algo falla

| Síntoma | Causa probable |
| --- | --- |
| `503 CUPO_NO_DISPONIBLE` | La función de cálculo no responde o no tiene `CADENA_CONEXION`; falta `FUNCION_CALCULADORA_CUPO` en la API; o el rol no tiene permiso (Terraform sin aplicar) |
| `422 CUPO_TARIFA_NO_RESUELTA` | La tienda no tiene flete configurado para Inter en `CostosTransporteTienda` |
| `403 TIENDA_NO_PERMITIDA` | Falta el paso 5.1 o el documento de cupo (5.2): «no tiene cupo configurado para la API» |
| `403 PRODUCTO_NO_PERMITIDO` / `422` por producto | El producto 1371 sigue inactivo (5.3) |
| `422 ORIGEN_NO_CONFIGURADO` | El proveedor del producto no tiene DANE o dirección |
| `401` / `403` sin cuerpo de la API | Token vencido, scope insuficiente o falta `x-api-key` |
| La guía se queda en `RECIBIDA` | El worker no recibe mensajes (revisar el disparador de la cola), el cliente está bloqueado, o `CADENA_CONEXION`/credenciales de Inter faltan en el worker |
| `ValidationException`/error al arrancar con «DocumentDB» o «bastión» | La guarda de entorno: una Lambda que no es de producción nunca se conecta a DocumentDB ni al puerto 27018; revisar `CADENA_CONEXION` y `ENTORNO` |
