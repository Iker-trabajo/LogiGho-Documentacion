## Autor: Iker Acevedo

Fecha creacion: 2026-09-21

Estado: desarrollo

## Infraestructura: Repositorios

**Ubicación:** `Infraestructura/Repositorio/`

---

## `UsuarioRepository` — roles de plataforma

**Colección:** `Users` (la misma de toda la plataforma, no una tabla propia de este
lambda).

```
ObtenerPorSubAsync(sub, ct)
  filtro = { "cognitoId": sub }
  (cognitoId es un arreglo en Users — un filtro directo hace match si "sub" es
  alguno de sus elementos, no hace falta $in ni $elemMatch)
  si no hay documento -> null
  devuelve UsuarioPlataforma { Email, Username, Roles }
```

Sin caché ni TTL: cada `solicitar-carga`/`confirmar-carga` consulta `Users` en vivo, así
que quitarle el rol a alguien en `Users` bloquea su siguiente petición inmediatamente,
sin esperar ninguna expiración de token ni de caché.

---

## `CargaRepository` — auditoría + candado de la carga

**Colección:** `CargasReportesAnalytics`. Usa `CargaId` también como `_id` — único
nativo, sin índice adicional que crear.

| Método | Qué hace |
| ------ | -------- |
| `InsertarAsync(carga)` | `InsertOneAsync`, `_id = carga.CargaId` |
| `ObtenerAsync(cargaId)` | `findOne` por `_id`, deserializa campo por campo (tolera documentos sin `Autor` en cargas creadas antes de agregarlo) |
| `AdquirirAsync(cargaId, propietario)` | Compare-and-set — ver abajo |
| `FinalizarAsync(cargaId, estado, hallazgos?, reporteId?, versionId?)` | `$set` de `Estado`, `FechaFinalizacion`, y opcionalmente `Hallazgos`/`ReporteIdPublicado`/`VersionIdPublicado`. Filtra por `_id + Estado=PENDIENTE + EnProcesamiento=true` — si no matchea, `409 CARGA_CAMBIO_DURANTE_CONFIRMACION` |
| `RegistrarErrorAsync(cargaId, codigo)` | `$set ErrorProcesamiento` — no toca `Estado`, la carga queda `PENDIENTE`/`EnProcesamiento=true` para conciliación manual |

### `AdquirirAsync` — el compare-and-set

```csharp
var filtro = new BsonDocument
{
    { "_id", cargaId }, { "IdentidadOrigen", propietario }, { "Estado", "PENDIENTE" },
    { "EnProcesamiento", new BsonDocument("$ne", true) }
};
var update = new BsonDocument("$set", new BsonDocument
{
    { "EnProcesamiento", true }, { "FechaInicioProcesamiento", DateTime.UtcNow.ToString("O") }
});
return (await collection.UpdateOneAsync(filtro, update)).ModifiedCount == 1;
```

Una sola operación atómica de Mongo decide si esta invocación es la que "gana" el
derecho a procesar la carga. Si `ModifiedCount != 1`, alguien más ya la tomó, ya no es
del dueño que llama, o ya no está `PENDIENTE` — sin distinguir cuál de esos casos fue,
`ConfirmarCargaUseCase` responde `409 CARGA_NO_PENDIENTE` en cualquiera.

---

## `ReporteAnalyticsRepository` — persistencia con versionado

**Colección:** `ReportesAnalytics`.

| Método | Qué hace |
| ------ | -------- |
| `ExisteAsync(reporteId)` | `Find(...).Limit(1).AnyAsync` |
| `PublicarAsync(reporte, nuevo)` | Insert si `nuevo`; si no, `$set` + `$push` atómico — ver abajo |

### Publicar una versión nueva de un reporte existente

```csharp
var filtro = new BsonDocument
{
    { "ReporteId", reporte.ReporteId },
    { "Versiones.VersionId", new BsonDocument("$ne", reporte.VersionActiva) }
};
var actualizacion = new BsonDocument { { "$set", cambios }, { "$push", new BsonDocument("Versiones", version) } };
var resultado = await collection.UpdateOneAsync(filtro, actualizacion);
if (resultado.MatchedCount != 1) throw new ApiException(409, "REPORTE_CAMBIO_DURANTE_CARGA");
```

El segundo término del filtro (`Versiones.VersionId $ne nuevaVersionId`) actúa como
compare-and-set: si el reporte cambió entre que se leyó (`ExisteAsync`, al principio del
flujo) y se escribe acá, o si por cualquier motivo esa versión ya estuviera presente,
`MatchedCount != 1` y se rechaza con `409` en vez de arriesgarse a corromper el array
`Versiones[]` con una escritura fuera de orden. `$set` + `$push` en la misma operación
garantiza que los metadatos editables y la versión nueva queden consistentes entre sí,
sin ventana donde uno se actualizó y el otro no.

---

## `MongoDatabase` — configuración de conexión

No es un repositorio propio, se configura en `Function.cs` (`CrearServicios()`):
`MongoClientSettings.FromConnectionString` sobre el valor **ya desencriptado** con
`EncripcionAES.DecryptAES256ECB` (ver
[servicios.md](servicios.md#decodificadorjwt-y-el-secreto-de-mongo)). TLS obligatorio
con validación de certificado (lanza al construir el `ServiceProvider` si
`AllowInsecureTls` es verdadero), `ReadPreference.Primary`, `WriteConcern.WMajority`,
`RetryWrites=false`, `ServerSelectionTimeout`/`ConnectTimeout` de 5s.

A diferencia de `MongoConexion` en `ApiLambdaDevolucionesMasivo`, acá no se fija
`MaxConnectionPoolSize` explícito — pendiente de revisar si este lambda necesita el
mismo acotamiento de pool bajo carga concurrente alta (ver la razón de ese ajuste en
[repositorios.md de DevolucionesMasivo](../../ApiLambdaDevolucionesMasivo/infraestructura/repositorios.md#mongoconexion--cliente-unico-por-contenedor)).
