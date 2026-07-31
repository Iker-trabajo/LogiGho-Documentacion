## Autor: Iker Acevedo

Fecha creacion: 2026-07-30  
Estado: Produccion

---

## Lambda: ApiLambdaSincronizacionSqlMongo

**Accionador:** Step Function (uso diario) | API Gateway (Sicronizacion via peticion HTTP)  
**AOT:** No

---

## ¿Qué hace?

Sincroniza datos desde Aurora **MySQL** (RDS) hacia **MongoDB/DocumentDB**, de forma **genérica**: la tabla origen, la colección destino, la columna llave y el modo (completo/incremental) llegan por parámetro en cada invocación, no hay tablas hardcodeadas en el código. Un mismo despliegue sirve para sincronizar cualquier tabla que se necesite, hoy o en el futuro, sin tocar código.

Soporta dos modos:

- **Completa**: vuelca la tabla entera, reemplazando el contenido de la colección destino.
- **Incremental**: trae solo los registros creados/modificados desde la última corrida exitosa, usando una columna de fecha de actualización.

Es de **solo lectura** sobre MySQL — nunca escribe, actualiza ni borra nada en el origen.

---

## Accionador


| Método/Origen | Ruta / Mecanismo                                       | Uso                                              | Auth                             |
| ------------- | ------------------------------------------------------ | ------------------------------------------------ | -------------------------------- |
| Step Function | Invocación directa de Lambda (`lambda:InvokeFunction`) | Caso regular: sincronización diaria automatizada | Rol IAM del state machine        |
| `POST`        | `/sincronizar`                                         | Sincronización manual via HTTP                   | Controles nativos de API Gateway |


> ⚠️ La Lambda **detecta sola** el origen de la invocación inspeccionando el evento crudo: si trae la propiedad `body`, lo trata como API Gateway; si no, como Step Function. No hace falta configurar nada distinto en el código para cada canal

---

## Request

### Vía API Gateway (Postman)

El evento llega envuelto en el formato estándar de API Gateway (`APIGatewayProxyRequest`); el JSON de negocio va **como string** dentro de `body`:

```json
{
  "resource": "/sincronizar",
  "path": "/sincronizar",
  "httpMethod": "POST",
  "headers": { "Content-Type": "application/json" },
  "body": "{\"TablaOrigen\":\"Clientes\",\"ColeccionDestino\":\"SQLClientes\",\"ColumnaLlavePrimaria\":\"IdCliente\",\"ColumnaFechaActualizacion\":\"FechaActualizacion\",\"SincronizacionCompleta\":false,\"TamanoBatch\":5000}",
  "isBase64Encoded": false
}
```

### Vía Step Function (invocación directa)

El evento **es** el JSON de negocio, sin sobre HTTP:

```json
{
  "TablaOrigen": "Clientes",
  "ColeccionDestino": "SQLClientes",
  "ColumnaLlavePrimaria": "IdCliente",
  "ColumnaFechaActualizacion": "FechaActualizacion",
  "SincronizacionCompleta": false,
  "TamanoBatch": 5000
}
```

### Campos del cuerpo de negocio (`PeticionSincronizacionDto`)


| Campo                       | Tipo      | Requerido                                | Default | Descripción                                                                                                                       |
| --------------------------- | --------- | ---------------------------------------- | ------- | --------------------------------------------------------------------------------------------------------------------------------- |
| `TablaOrigen`               | `string`  | Sí                                       | —       | Nombre exacto de la tabla en MySQL. Solo letras, números y `_` (protección SQL injection)                                         |
| `ColeccionDestino`          | `string`  | Sí                                       | —       | Nombre de la colección destino en Mongo. Mismo alfabeto restringido que `TablaOrigen`                                             |
| `ColumnaLlavePrimaria`      | `string`  | Sí                                       | —       | Columna usada como llave del documento (filtro de upsert en modo incremental). Debe existir en la tabla — se valida antes de leer |
| `ColumnaFechaActualizacion` | `string?` | Solo si `SincronizacionCompleta = false` | `null`  | Columna `DATETIME` que la Lambda usa como cursor incremental                                                                      |
| `SincronizacionCompleta`    | `bool`    | No                                       | `false` | `true` = carga íntegra (reemplaza destino). `false` = incremental                                                                 |
| `TamanoBatch`               | `int`     | No                                       | `5000`  | Filas por lote de inserción/upsert. Rango válido: `1`–`50000`                                                                     |


---

## Response

### Vía API Gateway

Siempre un `APIGatewayProxyResponse` con `statusCode` HTTP y el cuerpo de negocio serializado en `body`.

**Éxito (200):**

```json
{
  "statusCode": 200,
  "body": "{\"Error\":false,\"Mensaje\":\"Sincronización completada exitosamente.\",\"TablaOrigen\":\"Clientes\",\"ColeccionDestino\":\"SQLClientes\",\"RegistrosInsertados\":3,\"RegistrosActualizados\":1,\"TotalRegistrosProcesados\":4,\"TipoSincronizacion\":\"Incremental\",\"FechaEjecucion\":\"2026-07-30T18:32:10Z\",\"DuracionMs\":842}"
}
```

### Vía Step Function

En éxito, la Lambda devuelve el `RespuestaSincronizacionDto` **crudo** (sin sobre HTTP) como salida de la tarea — pasa directo al siguiente estado del state machine.

En error, la Lambda **lanza una excepción no controlada** en vez de devolver un objeto con `Error: true`. Es intencional: Step Function detecta el fallo de una tarea por la excepción, no por inspeccionar campos dentro de un JSON 200 — un `Catch`/`Retry` configurado en el state machine solo reacciona si la Lambda realmente falla.

### Campos de `RespuestaSincronizacionDto`


| Campo                      | Tipo       | Descripción                                                         |
| -------------------------- | ---------- | ------------------------------------------------------------------- |
| `Error`                    | `bool`     | `false` en éxito                                                    |
| `Mensaje`                  | `string`   | Descripción del resultado o del error                               |
| `TablaOrigen`              | `string`   | Eco de la tabla procesada                                           |
| `ColeccionDestino`         | `string`   | Eco de la colección procesada                                       |
| `RegistrosInsertados`      | `int`      | Documentos nuevos creados en Mongo                                  |
| `RegistrosActualizados`    | `int`      | Documentos existentes actualizados (solo aplica en incremental)     |
| `TotalRegistrosProcesados` | `int`      | `RegistrosInsertados + RegistrosActualizados`                       |
| `TipoSincronizacion`       | `string`   | `"Completa"` o `"Incremental"`                                      |
| `FechaEjecucion`           | `DateTime` | UTC, momento en que terminó la corrida                              |
| `DuracionMs`               | `long`     | Duración medida con `Stopwatch`, desde el inicio de `EjecutarAsync` |


### Errores (ruta API Gateway)


| Código | Cuándo                                                             | `Mensaje` ejemplo                                              |
| ------ | ------------------------------------------------------------------ | -------------------------------------------------------------- |
| `400`  | `body` vacío o JSON inválido                                       | "El cuerpo de la peticion esta vacio..."                       |
| `400`  | Campo obligatorio faltante                                         | "El campo TablaOrigen es obligatorio."                         |
| `400`  | Nombre con caracteres no permitidos (SQL injection)                | "...contiene caracteres no permitidos."                        |
| `400`  | Tabla fuera de la allowlist                                        | "La tabla '...' no está habilitada para sincronización."       |
| `400`  | `TamanoBatch` fuera de rango                                       | "TamanoBatch debe estar entre 1 y 50000."                      |
| `400`  | Columna llave no existe en la tabla                                | "La columna llave primaria '...' no existe en la tabla '...'." |
| `408`  | La ejecución se acercó al timeout de la Lambda (900s)              | "La sincronización superó el tiempo disponible..."             |
| `500`  | Error de conexión, MySQL, Mongo, o cualquier excepción no prevista | "Error interno del servidor: {detalle}"                        |


---

## Colección `ControlSincronizacion` (Mongo)

Colección interna de la Lambda — **no es datos de negocio**, es el histórico/cursor de cada sincronización. Se consulta antes de cada corrida incremental y se actualiza al final de cada corrida (exitosa o fallida).

**Clave:** compuesta por `TablaOrigen` + `ColeccionDestino` (no solo `TablaOrigen`). Motivo: la misma tabla MySQL puede sincronizarse hacia varias colecciones Mongo distintas, y cada destino necesita su propio cursor independiente — con clave simple, dos sincronizaciones de la misma tabla se pisaban la marca de agua.


| Campo                      | Tipo       | Descripción                                                                                                                                                                    |
| -------------------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `TablaOrigen`              | `string`   | Parte de la clave                                                                                                                                                              |
| `ColeccionDestino`         | `string`   | Parte de la clave                                                                                                                                                              |
| `UltimaSincronizacion`     | `DateTime` | Marca de agua del cursor. Es la hora del **servidor MySQL** (`UTC_TIMESTAMP(6)`), capturada **antes** de leer los datos — no la hora de la Lambda al terminar. Ver ADR punto 5 |
| `UltimoEstado`             | `string`   | `"Exitoso"` o `"Fallido"`                                                                                                                                                      |
| `RegistrosUltimaEjecucion` | `int`      | Total procesado en la última corrida exitosa                                                                                                                                   |
| `UltimoError`              | `string?`  | Mensaje de la última excepción, si `UltimoEstado = "Fallido"`. `null` en éxito                                                                                                 |
| `FechaRegistro`            | `DateTime` | UTC, cuándo se escribió este documento (auditoría)                                                                                                                             |


**Regla crítica:** si una corrida falla, el cursor (`UltimaSincronizacion`) **no avanza** — se conserva el valor anterior. Solo se escribe `"Exitoso"` cuando la sincronización completó sin excepciones. Esto evita que registros no sincronizados se pierdan silenciosamente.

---

## Flujo interno

```
Function.cs :: FunctionHandler(JsonElement, ILambdaContext)
  -> detecta origen (¿tiene "body"?)
     -> ManejarInvocacionApiGatewayAsync   (API Gateway: atrapa errores, responde con status code)
     -> ManejarInvocacionStepFunctionAsync (Step Function: deja propagar excepciones)
        -> ISincronizadorSqlMongo.EjecutarAsync(PeticionSincronizacionDto, CancellationToken)
           -> SincronizadorSqlMongo.EjecutarAsync
              1. ValidarPeticion            (campos, alfabeto anti-injection, allowlist, TamanoBatch)
              2. Abre conexión MySQL, fuerza sesión a UTC
              3. ObtenerInstanteServidorAsync   (marca de agua = hora del servidor MySQL, ANTES de leer)
              4. SincronizacionCompleta ? EjecutarSincronizacionCompletaAsync
                                         : EjecutarSincronizacionIncrementalAsync
              5. Guarda ControlSincronizacion (Exitoso + marca de agua) o (Fallido, sin mover el cursor)
```

### Sincronización completa

`SincronizadorSqlMongo::EjecutarSincronizacionCompletaAsync`

1. `SELECT * FROM {TablaOrigen}` completo.
2. Los documentos se insertan en una colección temporal `**{ColeccionDestino}__staging**` (no directo en el destino).
3. Al terminar, **swap atómico** vía comando Mongo `renameCollection` con `dropTarget: true` — reemplaza el destino por el staging en una sola operación.
4. **Por qué staging + swap:** si la lectura de MySQL falla a mitad de camino, los consumidores de la colección destino siguen viendo los datos completos de la corrida anterior hasta el instante exacto del swap. La alternativa (`DeleteMany` + insertar directo) deja el destino vacío o parcial si algo falla — inaceptable para consumidores en producción.
5. **Caso tabla vacía:** si la tabla origen tiene 0 filas, Mongo nunca llega a crear la colección de staging (no la crea hasta el primer `insert`). La Lambda detecta este caso y hace `DropCollection` directo sobre el destino, sin pasar por el swap.

### Sincronización incremental

`SincronizadorSqlMongo::EjecutarSincronizacionIncrementalAsync`

1. Consulta el cursor en `ControlSincronizacion` para el par (`TablaOrigen`, `ColeccionDestino`). Si nunca se sincronizó, arranca desde `DateTime.MinValue`.
2. `SELECT * FROM {TablaOrigen} WHERE {ColumnaFechaActualizacion} >= @UltimaFecha AND {ColumnaFechaActualizacion} < @InstanteCorte` — ventana **cerrada por abajo, abierta por arriba**. El límite superior evita arrastrar filas que se modifican *durante* la lectura (esas caen íntegras en la siguiente corrida).
3. Cada fila se aplica como `ReplaceOneModel` con `IsUpsert = true`, filtrando por `ColumnaLlavePrimaria` — inserta si no existe, reemplaza si existe.
4. El conteo de insertados/actualizados sale de `BulkWriteResult.Upserts` / `ModifiedCount` (no de `InsertedCount`, que siempre es 0 en operaciones de upsert).

### Mapeo de tipos MySQL → BSON

`SincronizadorSqlMongo::ConvertirABsonValue` — cubre enteros con/sin signo (incluyendo `BIGINT UNSIGNED`, que se guarda como `Decimal128` si excede `Int64.MaxValue`), decimales, fechas (siempre normalizadas a UTC), `TIME` (se guarda como string, Bson no tiene tipo equivalente), `GUID`, binarios y `NULL`.

---

## Dependencias externas


| Servicio                             | Uso                                                                              |
| ------------------------------------ | -------------------------------------------------------------------------------- |
| Aurora MySQL (RDS)                   | Origen de los datos, vía `MySqlConnector`                                        |
| MongoDB / DocumentDB                 | Destino de los datos + colección `ControlSincronizacion`                         |
| `Seguridad.Encripcion.EncripcionAES` | Descifrado AES-256-ECB de las cadenas de conexión (proyecto compartido del repo) |


---

## Variables de entorno


| Variable                   | Descripción                                                         | Obligatoria                |
| -------------------------- | ------------------------------------------------------------------- | -------------------------- |
| `CADENA_CONEXION_SQL`      | Cadena de conexión a Aurora MySQL, cifrada AES-256-ECB              | Sí (Release)               |
| `CADENA_CONEXION_MONGO`    | Cadena de conexión a Mongo/DocumentDB, cifrada AES-256-ECB          | Sí (Release)               |
| `DATABASE_NAME_MONGO`      | Base de datos Mongo destino                                         | No (default `"LogighoDB"`) |
| `SINCRO_TABLAS_PERMITIDAS` | CSV de tablas habilitadas para sincronizar. Vacía = sin restricción | No                         |


> En compilación **Debug** (local), estas cuatro variables no se leen — las credenciales están hardcodeadas contra los servicios locales (Docker MySQL / Mongo Compass) directo en `ConfiguracionEntorno.Cargar()`, mismo patrón que el resto de lambdas del repo.

---

## Configuración Lambda


| Parámetro    | Valor                                   |
| ------------ | --------------------------------------- |
| Runtime      | `dotnet8`                               |
| Handler      | `ApiLambdaSincronizacionSqlMongo`       |
| Framework    | `net8.0`                                |
| Memory       | `3072 MB`                               |
| Timeout      | `900 segundos` maximo permitido por AWS |
| Architecture | `x86_64`                                |
| Package      | `Zip` · `--self-contained true`         |


---

## Arquitectura

```
ApiLambdaSincronizacionSqlMongo/
├── Function.cs                          ← Entry point, detección de origen (API Gateway / Step Function)
├── Aplicacion/
│   ├── DTO/            (PeticionSincronizacionDto, RespuestaSincronizacionDto)
│   └── Interfaces/      (ISincronizadorSqlMongo, IControlSincronizacionRepositorio)
├── Dominio/
│   └── ControlSincronizacion.cs         ← Documento de cursor/histórico
└── Infraestructura/
    ├── SincronizadorSqlMongo.cs         ← Caso de uso: lectura MySQL, escritura Mongo, validaciones
    ├── ControlSincronizacionRepositorio.cs
    └── ConfiguracionEntorno.cs          ← Carga de credenciales (Debug hardcode / Release descifrado)

Dependencia de proyecto:
LambdasLogiGho.Infraestructura/Seguridad/Encripcion.AES/EncripcionAES.cs  ← AES-256-ECB
```

---

## Historial de cambios


| Fecha      | Autor        | Cambio                                                                                                                                                                                                                           |
| ---------- | ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 2026-07-30 | Iker Acevedo | Lambda reescrita completa: cambio de motor SQL Server→MySQL, dual invocación API Gateway/Step Function, staging+swap, fix ventana incremental, allowlist de tablas, cursor con clave compuesta. Desplegada y validada en preprod |


---

## Observaciones

> Deuda técnica, comportamientos especiales, decisiones no obvias del código. Ver también el ADR de esta lambda para el detalle de cada decisión.
>
>

- **Pendiente:** Hacer upgrade a NET10 pero hubieron muchos problemas a la hora de desplegar en NET10 se bajo a NET8
- Es de **solo lectura** sobre MySQL cualquier futuro cambio que necesite escribir en el origen requiere una revisión de diseño aparte, no es una extensión trivial de esta lambda.

