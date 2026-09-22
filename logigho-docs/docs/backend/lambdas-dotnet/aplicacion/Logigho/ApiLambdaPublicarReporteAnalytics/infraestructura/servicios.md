## Autor: Iker Acevedo

Fecha creacion: 2026-09-21

Estado: desarrollo

## Infraestructura: Servicios

**Ubicación:** `Infraestructura/Servicios/`

---

## `DecodificadorJwt` y el secreto de Mongo

```csharp
public static class DecodificadorJwt
{
    public static string? ExtraerSub(string? jwt);
}
```

Decodifica el segmento payload del JWT (base64url) y lee el claim `sub` —
**sin verificar firma, issuer, audience ni expiración**. Mismo patrón que
`JwtClaimsHelper` en `ApiLambdaConsultarPedidos` (`ServicioPublicoLogigho`). No es un
descuido: el API Gateway real no tiene ningún Authorizer nativo en ninguna ruta de la
plataforma (confirmado con `aws apigatewayv2 get-routes`, todo `AuthorizationType:
NONE`), así que la seguridad real no vive acá — vive en que el `sub` exista en `Users`
con uno de los roles permitidos (`AutorizacionUsuario`, ver
[modelos.md](../dominio/modelos.md#identidadcaller)).

Devuelve `null` (no lanza) ante cualquier JWT malformado — el llamador (`Function.cs`)
lo trata como `401 IDENTIDAD_AUSENTE` sin distinguir "no mandó token" de "mandó un token
roto".

**Sobre `MONGODB_CONNECTION_STRING`:** no es responsabilidad de esta clase, pero es el
mismo estándar de cifrado. `Function.cs` desencripta el valor del entorno con
`Seguridad.Encripcion.EncripcionAES.DecryptAES256ECB` antes de pasarlo a
`MongoClientSettings.FromConnectionString` — la misma clase AES256-ECB que usa el resto
del repo para todo secreto sensible en variables de entorno (contraseñas, connection
strings, credenciales de transportadoras). El proyecto referencia
`LambdasLogiGho.Infraestructura/Seguridad/Seguridad.csproj` para esto, igual que
`ApiLambdaDevolucionesMasivo`.

---

## `PresignedUrlService`

```csharp
public string CrearUrl(string cargaId, DateTimeOffset expiraEn) => s3.GetPreSignedURL(new GetPreSignedUrlRequest
{
    BucketName = opciones.Bucket, Key = ReporteStorage.ClaveStaging(cargaId),
    Verb = HttpVerb.PUT, Expires = expiraEn.UtcDateTime, Protocol = Protocol.HTTPS
});
```

Firma **solo** una URL `PUT` para el objeto de staging correspondiente al `CargaId`
dado — nunca para una key arbitraria que el cliente elija. Usa las credenciales del rol
de ejecución de la Lambda; no hay access keys en código ni en configuración.

La firma SigV4 incluye el verbo HTTP: pegar esta URL en un navegador hace `GET` y da
`SignatureDoesNotMatch` — comportamiento esperado, no un bug (se confirmó en pruebas
reales contra PreProd).

---

## `ReporteStorage`

Acceso a S3 acotado a cuatro operaciones, todas con límites reales de bytes — nunca se
confía en `Content-Length` ni en el tamaño que declaró el cliente.

| Método | Qué hace |
| ------ | -------- |
| `DescargarStagingAsync(cargaId)` | Lee `staging/reportes-analytics/{cargaId}/original.html` con límite real de 15 MB+1; calcula SHA-256 sobre lo leído |
| `GuardarVersionAsync(key, base64Gzip)` | `PutObject` con `IfNoneMatch: "*"` (nunca sobreescribe una key existente) + `ChecksumAlgorithm.SHA256` de transferencia |
| `LeerVersionAsync(key)` | Relee una key ya escrita (para la segunda verificación de integridad en `ConfirmarCargaUseCase`), límite 22 MB (gzip+base64 pueden pesar más que el original) |
| `BorrarStagingAsync(cargaId)` | `DeleteObject` del staging, llamado siempre en el `finally` del caso de uso |

### Por qué nunca confía en `Content-Length`

```csharp
var leidos = await objeto.ResponseStream.ReadAsync(buffer.AsMemory(0,
    (int)Math.Min(buffer.Length, limite + 1L - salida.Length)), ct);
...
if (salida.Length > limite)
    throw new ValidacionException([new("TAMANO_EXCEDIDO", "...")]);
```

Se lee el stream directamente, nunca más de `límite + 1` bytes en total, sin importar lo
que diga el header. Un `Content-Length` falso o un objeto más grande de lo declarado no
puede hacer que el proceso cargue de más en memoria — el corte pasa por bytes realmente
leídos.

### Checksums multipart

```csharp
if (!string.IsNullOrEmpty(objeto.ChecksumSHA256) && !objeto.ChecksumSHA256.Contains('-') && ...)
```

Un objeto subido en múltiples partes trae un checksum con sufijo `-N` que **no** es el
SHA-256 del objeto completo — se ignora explícitamente esa forma para no comparar contra
un valor que nunca va a coincidir por definición, no porque el objeto esté corrupto.

---

## `ReporteValidadorService`

**Interfaz:** [`IReporteValidadorService`](../ApiLambdaPublicarReporteAnalytics.md#validación-de-contenido-las-8-reglas)
— acá vive la implementación completa de las 8 reglas (ver la tabla en el documento
principal de la lambda).

Detalles de implementación no obvios:

- **Regex compiladas con `RegexOptions.ECMAScript`**: mantiene `\b` y `\s` compatibles
  con la semántica de JavaScript (de donde vienen las reglas originales de Angular), en
  vez de la semántica por defecto de .NET.
- **`RegexOptions.CultureInvariant`**: evita que un cambio de cultura del host altere el
  resultado de un match — las reglas deben comportarse igual sin importar dónde corre el
  contenedor Lambda.
- **Timeout de 1 segundo por evaluación** de regex (`new Regex(..., TimeSpan.FromSeconds(1))`),
  más un presupuesto total de 10 segundos para todo el escaneo (`Stopwatch` explícito) —
  ante contenido adversarial diseñado para forzar backtracking catastrófico, el análisis
  se corta y rechaza (`VALIDACION_AGOTADA`) en vez de colgar la invocación.
- **Escaneo por línea (`AsSpan().Split('\n')`) + escaneo del documento completo**: el
  primero da número de línea real para el hallazgo; el segundo, con las mismas 8 reglas,
  agarra tokens o atributos partidos entre saltos de línea que el escaneo por línea no
  vería (por ejemplo, un `src="` en una línea y la URL en la siguiente).
- **Máximo 1000 hallazgos** (`LIMITE_HALLAZGOS`): un archivo con miles de violaciones no
  genera una respuesta de tamaño ilimitado.
- El recorrido usa `Span<char>` sobre el string original, nunca un array de millones de
  substrings — importa para archivos de hasta 15 MB con muchos saltos de línea.

---

## `CompresionGzipService`

```csharp
public interface ICompresionGzipService
{
    byte[] Comprimir(byte[] contenido);
    byte[] DescomprimirVerificado(byte[] gzip, string sha256);
}
```

Gzip RFC1952 estándar (`System.IO.Compression.GZipStream`), compatible con lo que
`pako.ungzip` del front espera al leer una versión ya publicada.

`DescomprimirVerificado` no solo descomprime: recalcula el SHA-256 de la salida y lo
compara contra el hash esperado con `CryptographicOperations.FixedTimeEquals`
(comparación en tiempo constante, evita timing attacks aunque acá el hash no es un
secreto — es el estándar del repo para toda comparación de hash). También corta la
descompresión si el resultado supera el límite de 15 MB, para no gastar memoria
descomprimiendo un gzip bomb.

---

## `OpcionesReporte`

```csharp
public sealed record OpcionesReporte(string Bucket);
```

Envoltorio simple del bucket de S3 leído del entorno (`REPORTES_S3_BUCKET`), inyectado
como singleton — evita leer la variable de entorno repetida en cada servicio que
necesita el nombre del bucket.
