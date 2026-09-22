## Autor: Iker Acevedo

Fecha creacion: 2026-09-21

Estado: desarrollo

## Caso de uso: ConfirmarCargaUseCase

**Ubicación:** `Aplicacion/CasosUso/ConfirmarCargaUseCase.cs`
**Lo invoca:** `Function.cs` — `POST /reportes-analytics/confirmar-carga/{cargaId}`

---

## ¿Qué hace?

El corazón del módulo. Toma una carga `PENDIENTE` ya creada por
[`SolicitarCargaUseCase`](solicitar-carga-usecase.md), descarga el HTML que el cliente ya
subió a S3, lo valida de verdad (tamaño real, UTF-8, las 8 reglas de seguridad),
lo comprime, lo publica en la key definitiva, y actualiza `ReportesAnalytics` — todo con
verificación de integridad en cada paso.

```
EjecutarAsync(sub, cargaId, leerRequest, logger, ct)
  1. AutorizacionUsuario.ExigirRolAsync(sub, usuarios, ct)          401
  2. Guid.TryParseExact(cargaId)                                    400 CARGA_ID_INVALIDO
  3. cargas.ObtenerAsync(cargaId)                                   404 CARGA_NO_EXISTE
  4. carga.IdentidadOrigen != caller.Sub                             403 CARGA_AJENA
  5. carga.Estado != PENDIENTE || !cargas.AdquirirAsync(...)         409 CARGA_NO_PENDIENTE
     (compare-and-set: dueno + PENDIENTE + EnProcesamiento != true)
  --- a partir de aca, esta carga es exclusiva de esta invocacion ---
  try
    6. ahora >= FechaSolicitud + 10min                               410 CARGA_EXPIRADA
    7. leerRequest() + ValidarMetadata()                             400 METADATOS_INVALIDOS
    8. si carga.ReporteId, reportes.ExisteAsync                      404 REPORTE_NO_EXISTE
    9. storage.DescargarStagingAsync(cargaId)                        limite real de bytes
   10. objeto.Bytes.Length > 15 MiB -> ValidacionException           422 TAMANO_EXCEDIDO
   11. SHA-256 sobre los bytes crudos
   12. hash != objeto.Sha256 (checksum de transferencia S3)          500 INTEGRIDAD_INVALIDA
   13. UTF8Encoding(throwOnInvalid).GetString                        422 UTF8_INVALIDO
   14. validador.Validar(html, nombreArchivo)                        422 con Hallazgos
   15. gzip = compresion.Comprimir(bytes)
       compresion.DescomprimirVerificado(gzip, hash)                 roundtrip local
   16. key = reportes-analytics/{reporteId}/{versionId}.html.gz
       storage.GuardarVersionAsync(key, base64(gzip))                If-None-Match:*
   17. relee esa MISMA key recien escrita
       compresion.DescomprimirVerificado(releido, hash)               segunda verificacion
   18. arma VersionReporte + ReporteAnalytics
   19. reportes.PublicarAsync(reporte, nuevo: carga.ReporteId==null)  $push atomico
   20. cargas.FinalizarAsync(cargaId, COMPLETADA, ...)
   21. devuelve ConfirmarCargaResponse(true, ReporteId, VersionId)
  catch ValidacionException
    cargas.FinalizarAsync(cargaId, RECHAZADA, hallazgos, ...)
    devuelve ConfirmarCargaResponse(false, Hallazgos: ...)            422, no relanza
  catch ApiException con StatusCode < 500
    cargas.FinalizarAsync(cargaId, RECHAZADA, ...)
    relanza                                                           lo maneja Function.cs
  catch cualquier otra excepcion
    cargas.RegistrarErrorAsync(cargaId, REVISION_REQUERIDA)           fallo ambiguo
    relanza
  finally
    storage.BorrarStagingAsync(cargaId)                                SIEMPRE, con token propio
    si falla el borrado -> 503 LIMPIEZA_PENDIENTE
```

---

## Por qué la toma de la carga es un compare-and-set

```csharp
if (carga.Estado != "PENDIENTE" || !await cargas.AdquirirAsync(cargaId, caller.Sub, ct))
    throw new ApiException(409, "CARGA_NO_PENDIENTE");
```

`AdquirirAsync` (`CargaRepository`) hace un único `UpdateOne` con filtro
`_id == cargaId AND IdentidadOrigen == propietario AND Estado == PENDIENTE AND EnProcesamiento != true`.
Si dos peticiones de confirmación llegan casi al mismo tiempo para la misma carga (doble
click, reintento de red del cliente), solo una consigue `ModifiedCount == 1` — la otra
recibe `409` sin haber tocado S3 ni Mongo más allá de esa comparación. Sin esto, dos
invocaciones concurrentes podrían pisarse la limpieza del staging o publicar la misma
versión dos veces.

---

## Por qué se verifica el hash tres veces

1. **Al descargar de staging**: contra el checksum de transferencia que S3 devuelve
   (`ChecksumSHA256`), para detectar corrupción en la bajada.
2. **Al comprimir localmente**: `DescomprimirVerificado(gzip, hash)` descomprime el gzip
   recién generado y confirma que da exactamente los mismos bytes — antes de gastar una
   escritura a S3 con algo potencialmente corrupto.
3. **Al releer la versión ya subida**: se vuelve a descargar la key definitiva
   inmediatamente después de escribirla y se descomprime de nuevo — para confirmar que
   lo que quedó persistido en S3 es exactamente lo que se pretendía, no una escritura
   parcial o corrompida en tránsito.

Ninguna de las tres reemplaza a las otras: cada una cubre un tramo distinto de la
cadena (bajada, procesamiento local, subida).

---

## Por qué la limpieza de staging corre en un `finally` con su propio token

```csharp
finally
{
    using var limpieza = new CancellationTokenSource(TimeSpan.FromSeconds(4));
    try { await storage.BorrarStagingAsync(cargaId, limpieza.Token); }
    catch { throw new ApiException(503, "LIMPIEZA_PENDIENTE"); }
}
```

El token de cancelación general de la petición puede estar ya vencido o cancelado
cuando se llega al `finally` (por ejemplo, si el cliente cortó la conexión). Usar un
`CancellationTokenSource` propio de 4 segundos garantiza que el intento de borrado
siempre corra, independiente de qué pasó antes. Si el borrado falla, se responde `503
LIMPIEZA_PENDIENTE` — pero la publicación ya puede haber quedado `COMPLETADA`: un 503 acá
no significa que la operación de negocio falló, solo que quedó basura en staging por
limpiar.

---

## Fallos ambiguos: por qué no se reintenta sola

```csharp
catch (Exception ex)
{
    ...
    await cargas.RegistrarErrorAsync(cargaId, "REVISION_REQUERIDA", registro.Token);
    throw;
}
```

Una excepción no prevista entre el `PutObject` a S3 y el `$push` en Mongo deja abierta
la duda de si el objeto quedó escrito pero el documento no, o viceversa. Reintentar
automáticamente ahí arriesga publicar la misma versión dos veces o dejar un objeto
huérfano sin registro. En vez de eso, la carga queda `PENDIENTE` con
`EnProcesamiento=true` y `ErrorProcesamiento=REVISION_REQUERIDA` — visible para
conciliación manual, sin bloquear otras cargas del mismo usuario.

---

## Observaciones

- `ValidarMetadata` corre **después** de descargar el staging, no antes — se prioriza
  fallar por contenido (lo que más tiempo de CPU cuesta) solo si los metadatos ya son
  válidos, evitando gastar la descarga de S3 en una petición que de todos modos iba a
  rechazarse por metadatos.
- El `Estado` final de una carga rechazada por contenido es `RECHAZADA`, igual que una
  rechazada por `ApiException` de negocio — no se distingue en el campo `Estado`, solo en
  `Hallazgos` (presente) vs. ausente.
- Ver [dominio/modelos.md](../dominio/modelos.md) para el detalle completo de
  `CargaReporteAnalytics`, `ReporteAnalytics` y `VersionReporte`.
