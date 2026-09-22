## Autor: Iker Acevedo

Fecha creacion: 2026-09-21

Estado: desarrollo

## Caso de uso: SolicitarCargaUseCase

**Ubicación:** `Aplicacion/CasosUso/SolicitarCargaUseCase.cs`
**Lo invoca:** `Function.cs` — `POST /reportes-analytics/solicitar-carga`

---

## ¿Qué hace?

Paso 1 del flujo de publicación. Autoriza, valida la *declaración* del archivo (nombre y
tamaño — no el contenido real todavía, eso pasa en
[`ConfirmarCargaUseCase`](confirmar-carga-usecase.md)), registra la intención de carga en
Mongo y devuelve una URL prefirmada de S3 para que el cliente suba el HTML directo,
sin pasar por esta Lambda.

```
EjecutarAsync(sub, leerRequest, logger, ct)
  1. AutorizacionUsuario.ExigirRolAsync(sub, usuarios, ct)      401 si no hay rol
  2. leerRequest()                                              deserializa el body
  3. validador.ValidarDeclaracion(NombreArchivo, TamanoDeclaradoBytes)
     si hay hallazgos -> throw ValidacionException              422, la maneja Function.cs
  4. si viene ReporteId:
       Guid.TryParseExact                                       400 REPORTE_ID_INVALIDO si no
       reportes.ExisteAsync(reporteId)                          404 REPORTE_NO_EXISTE si no
  5. crea CargaReporteAnalytics { CargaId=Guid nuevo, Estado=PENDIENTE, ... }
     cargas.InsertarAsync(carga)
  6. expiraEn = ahora + 10 minutos
     urls.CrearUrl(carga.CargaId, expiraEn)                     firma PUT de staging
  7. devuelve SolicitarCargaResponse(CargaId, UrlSubida, ExpiraEn)
```

El orden importa: se autoriza **antes** de leer o validar nada del body — si el llamador
no tiene permiso, no tiene sentido gastar trabajo en deserializar o validar.

---

## Por qué valida la declaración acá, y no espera a `confirmar-carga`

Rechazar un nombre de archivo sin extensión `.html`, o un tamaño ya evidentemente fuera
de rango, antes de crear el registro de carga y firmar una URL, evita generar objetos de
staging huérfanos y cargas `PENDIENTE` que nunca se van a poder completar. Es la misma
idea de "cortar temprano lo barato" que usa el resto del repo (ver por ejemplo
`IniciarJobUseCase` en `ApiLambdaDevolucionesMasivo`).

La validación de contenido real (las 8 reglas de seguridad, tamaño real de bytes,
UTF-8) no puede pasar acá porque en este punto el archivo todavía no existe en S3 — el
cliente recién va a subirlo después de recibir la URL.

---

## Por qué `ReporteId` se valida acá y no se asume

```csharp
if (!Guid.TryParseExact(request.ReporteId, "D", out var id)) throw new ApiException(400, "REPORTE_ID_INVALIDO");
reporteId = id.ToString("D");
if (!await reportes.ExisteAsync(reporteId, ct)) throw new ApiException(404, "REPORTE_NO_EXISTE");
```

Confirmar que el `ReporteId` exista **antes** de crear la carga evita que una carga con
un ID inexistente llegue viva hasta `confirmar-carga` y recién ahí falle — mejor decirle
al cliente de inmediato que ese reporte no existe, sin gastar el ciclo completo de
subida a S3 primero.

---

## Auditoría: `Autor` en la carga

```csharp
var carga = new CargaReporteAnalytics
{
    CargaId = Guid.NewGuid().ToString("D"), ReporteId = reporteId, IdentidadOrigen = caller.Sub,
    Autor = caller.Autor, NombreArchivoDeclarado = request.NombreArchivo,
    TamanoDeclaradoBytes = request.TamanoDeclaradoBytes, FechaSolicitud = ahora
};
```

`IdentidadOrigen` guarda el `sub` (UUID de Cognito, lo que se usa para las comparaciones
de dueño en `ConfirmarCargaUseCase`). `Autor` guarda el email/username legible que ya
resolvió `AutorizacionUsuario` — antes de este campo, auditar quién intentó una carga
rechazada exigía cruzar manualmente el `sub` contra `Users`.

---

## Observaciones

- La URL prefirmada expira en **10 minutos**, el mismo plazo que usa
  `ConfirmarCargaUseCase` para considerar una carga vencida (`CARGA_EXPIRADA`) — ambos
  plazos están atados a `carga.FechaSolicitud`, no son independientes.
- `IPresignedUrlService.CrearUrl` firma **solo** el objeto de staging correspondiente a
  ese `CargaId` — no puede usarse para escribir en ninguna otra key.
