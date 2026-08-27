## Autor: Iker Acevedo

Fecha creacion: 2026-08-26

Estado: produccion

## Modelos: ingreso-devoluciones.models.ts

**Ubicación:** `src/app/views/logistica/ingreso-devoluciones/models/ingreso-devoluciones.models.ts`

---

## Constantes de operación

| Constante | Valor | Para qué |
| --------- | ----- | -------- |
| `MAXIMO_GUIAS_POR_LOTE` | 3000 | Debe coincidir con `JobDevolucion.MaximoGuiasPorJob` del backend |
| `INTERVALO_POLLING_MS` | 2500 | Cada cuánto se consulta `/estado` mientras el lote está abierto |
| `ESPERA_ESCANEO_MS` | 3000 | Silencio del escáner que dispara el envío del buffer de pistoleo |
| `TOPE_BUFFER_ESCANEO` | 25 | Guías acumuladas antes de enviar sin esperar el silencio |
| `TOLERANCIA_LATIDO_MS` | 2 minutos | Replica el mismo umbral que usa `ConsultarJobUseCase` para `WorkerDetenido` — solo para explicarle el plazo al operario, la verdad la dice siempre el campo `WorkerDetenido` de la respuesta |
| `ENDPOINT_INICIAR`, `ENDPOINT_ESTADO` | `devolucionesMasivo/iniciar`, `devolucionesMasivo/estado` | Rutas de la API de lambdas |
| `COLECCION_JOBS`, `COLECCION_INVENTARIO` | `JobsDevolucion`, `InventarioDevolucion` | Para las consultas vía `metodoGenerico` |

---

## Enums del dominio (como texto, no como número)

```typescript
type EstadoJob = 'Pendiente' | 'EnProceso' | 'Completado' | 'Fallido';
type EstadoResolucion = 'YaProcesada' | 'ResueltaDirecta' | 'ResueltaInter' | 'Rechazada';
type MotivoRechazo = 'GuiaNoExisteEnSistema' | 'OriginalNoExisteEnSistema'
    | 'SinGuiaOriginalEnRespuesta' | 'PedidoAnulado' | 'SinPermisoTienda' | 'ErrorConsultaExterna';
```

Coinciden 1:1 con los enums de C# del backend (`EstadoJob`, `EstadoResolucion`, `MotivoRechazo`). Viajan como texto en el JSON porque el backend registra `JsonStringEnumConverter` en `RespuestaHttp` — sin eso, la API los mandaría como número mientras Mongo (leído vía `metodoGenerico`) los guarda como texto, y el front recibiría dos formas distintas del mismo dato según la fuente.

---

## Contratos de la API de lambdas

### `IniciarLoteResponse` — `POST /iniciar`

```typescript
{ Error: boolean; Mensaje?: string; JobId?: string; GuiasAceptadas: number; GuiasDescartadas: number; }
```

### `GuiaProcesada`

Espejo del `GuiaProcesada` de C#. Los 4 flags booleanos del final (`FueResuelta`, `EsRecienResuelta`, `RequiereActualizarInventario`, `MereceReintentoConOtraTransportadora`) son propiedades **calculadas** en C# — llegan por la API pero **no** se persisten en Mongo, así que al leer desde `InventarioDevolucion` vienen ausentes.

⚠️ **Ojo con el nombre exacto:** el campo real del backend es `EsRecienResuelta` (sin "te"). Un typo en este archivo (`EsRecienteResuelta`) dejaría el campo siempre `undefined` sin ningún error de compilación — TypeScript no puede detectar un nombre de propiedad opcional mal escrito contra un JSON externo.

### `EstadoLoteResponse` — `GET /estado`

```typescript
{
  Error, Mensaje?, JobId, Estado: EstadoJob,
  GuiasTotal, GuiasProcesadas, GuiasPendientes,
  Resueltas, Rechazadas, YaProcesadas,        // los 3 resultados, nunca sumados en la UI
  Porcentaje,
  FechaInicio, FechaUltimaActividad, FechaFin?,
  WorkerDetenido: boolean,
  MensajeError?,
  Detalle?: GuiaProcesada[]                    // solo con &detalle=true
}
```

`YaProcesadas` **no es un error** — es la idempotencia funcionando: la guía ya se había resuelto en una corrida anterior y no se volvió a tocar.

---

## Documentos crudos de Mongo (`metodoGenerico`)

### `JobDevolucionDoc`

Igual forma que `EstadoLoteResponse` pero con `_id` en vez de `JobId`, y `Rechazadas: GuiaProcesada[]` completo (no solo contadores) — es el documento tal cual vive en `JobsDevolucion`. `GuiasPendientes` no tiene detalle de las resueltas: ese vive en `InventarioDevolucion`.

### `InventarioDevolucionDoc`

Una fila por guía procesada.

⚠️ **`MotivoRechazo` puede llegar con el texto literal `"BsonNull"`** en las filas resueltas — es un artefacto del backend (`?? BsonNull.Value.ToString()` en `DevolucionRepository`, corregido después a `BsonNull.Value` directo, pero los documentos ya escritos con el texto viejo persisten). `motivoDesdeTexto()` en `rules.ts` normaliza ese caso tratándolo como ausencia de motivo.

⚠️ **El campo `'Observaciones Guia'` lleva un espacio en el nombre**, tal cual lo escribe el backend — hay que acceder con notación de índice (`doc['Observaciones Guia']`), no con punto.

---

## Catálogo de motivos — la traducción de enum técnico a lenguaje de bodega

`CATALOGO_MOTIVOS: Record<MotivoRechazo, MotivoInfo>` — por cada uno de los 6 motivos, define:

| Campo | Qué es |
| ----- | ------ |
| `titulo` | Nombre humano corto ("Guía huérfana", no "OriginalNoExisteEnSistema") |
| `explicacion` | Qué pasó, en una frase, sin jerga técnica |
| `accion` | Qué hace el operario con la caja física que tiene en la mano |
| `severidad` | `'error'` \| `'advertencia'` \| `'info'` |
| `familia` | `'dato'` \| `'negocio'` \| `'tecnico'` — de quién es el problema |
| `icono` | Clase de FontAwesome |
| `reintentable` | Si vale la pena reintentarla en un lote nuevo |

**Las 3 familias, y por qué importan:** agrupar los 6 motivos como una lista plana de errores rojos no le dice nada al operario sobre qué hacer. `dato` (el operario puede corregir), `negocio` (requiere que alguien más actúe), `tecnico` (se resuelve reintentando) — es el criterio que usa `resumenPorFamilia()` en `rules.ts` para agrupar el resultado final.

`MOTIVO_DESCONOCIDO`: si el backend algún día manda un motivo que este catálogo no conoce (alguien agregó un valor al enum de C# y se olvidó de actualizar acá), `infoMotivo()` devuelve esta ficha genérica en vez de romper el render con `undefined`.

---

## Modelos propios del front (no existen en el backend)

| Modelo | Para qué |
| ------ | -------- |
| `LoteHistorial` | Fila normalizada de la tabla de historial — incluye `idLote` (derivado, ver `derivarIdLote`) y `detalleRechazadas` (copiado directo de `JobDevolucionDoc.Rechazadas`, sin volver a pedirle nada a la API) |
| `EstadoEscaneo`, `GuiaEscaneada` | Estado de una guía dentro del buffer de pistoleo: `en-cola` → `enviando` → `resuelta`/`ya-registrada`/`rechazada`, o `duplicada` si se repite en la misma sesión |
| `GuiasDepuradas` | Resultado de limpiar la columna de guías de un archivo: `guias`, `vacias`, `duplicadas`, `excedeTope` |
