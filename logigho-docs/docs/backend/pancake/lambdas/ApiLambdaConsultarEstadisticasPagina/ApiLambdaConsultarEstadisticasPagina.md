## Autor: Iker Acevedo

Fecha creacion: 2026-07-27

Estado: produccion

## Lambda: ApiLambdaConsultarEstadisticasPagina

**Accionador:** API Gateway — `POST`

**AOT:** No

**Tipo:** Endpoint on-demand, Cold Start

**Url servicio prod:** [https://7elgyten88.execute-api.us-east-1.amazonaws.com/Produccion/obtenerDatosPaginasPancake](https://7elgyten88.execute-api.us-east-1.amazonaws.com/Produccion)

**Url servicio preprod:** [https://g3iz6qk3f0.execute-api.us-east-1.amazonaws.com/obtenerDatosPaginasPancake](https://g3iz6qk3f0.execute-api.us-east-1.amazonaws.com/obtenerDatosPaginasPancake)

---

## ¿Qué hace?

Expone un **endpoint HTTP** para consultar, **al instante y sin persistir**, las estadísticas por campaña de **una** página de Pancake. Pensado para cuando alguien duda de una pagina y quiere ver los números **frescos** tal cual como los muestra pancake, sin esperar a la corrida agendada.

Replica el comportamiento de Pancake: se puede pedir **hoy** (00:00 → ahora), **ayer** (día completo) o un **rango personalizado**. El back calcula la ventana en hora Colombia (-05:00) y llama a la API pública de Pancake. **No escribe en ninguna base de datos** — solo devuelve el resultado en el response.

---

## Accionador


| Método | Ruta                                        | Autenticación                                     |
| ------ | ------------------------------------------- | ------------------------------------------------- |
| `POST` | API Gateway — `/obtenerDatosPaginasPancake` | JWT validado por el **authorizer** de API Gateway |


---

## Request

Body JSON:

```json
{
  "paginaId": "waba_1239915952528289",
  "rango": "ayer"
}
```


| Campo      | Tipo     | Requerido               | Descripción                                                                                                                  |
| ---------- | -------- | ----------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `paginaId` | `string` | Sí                      | Id de la página (campo `paginaId` en `PancakePaginas`)                                                                       |
| `rango`    | `string` | Sí                      | `"hoy"`, `"ayer"` o `"personalizado"``Enum con 3 opciones si se selecciona personalizado tiene que inclur el desde / hasta` |
| `desde`    | `string` | Solo si `personalizado` | Inicio, ISO `yyyy-MM-dd HH:mm:ss` (hora Colombia)                                                                            |
| `hasta`    | `string` | Solo si `personalizado` | Fin, ISO `yyyy-MM-dd HH:mm:ss` (hora Colombia)                                                                               |


### Valores de `rango`


| `rango`         | Ventana resultante                        | Uso                     |
| --------------- | ----------------------------------------- | ----------------------- |
| `hoy`           | Hoy 00:00:00 → **ahora**                  | Foto del día en curso   |
| `ayer`          | Ayer 00:00:00 → 23:59:59                  | Día anterior completo   |
| `personalizado` | `desde` → `hasta` (exactos, admite horas) | Cualquier rango puntual |


### Ejemplo personalizado

```json
{
  "paginaId": "waba_1239915952528289",
  "rango": "personalizado",
  "desde": "2026-07-08 06:00:00",
  "hasta": "2026-07-09 14:59:59"
}
```

---

## Response

### Exitoso (200)

```json
{
  "paginaId": "waba_1239915952528289",
  "rango": "ayer",
  "fechaConsultada": "2026-07-26",
  "since": 1784005200,
  "until": 1784091599,
  "totalCampanias": 2,
  "campanias": [
    {
      "campañaId": "52557952887776",
      "nombre": "Cam 2 _ 16/07",
      "moneda": "COP",
      "gasto": "1588598",
      "cpm": "...",
      "alcance": "...",
      "impresiones": "252146"
    }
  ],
  "error": false,
  "mensaje": null
}
```

Se devuelven `since`/`until` (Unix UTC) para transparencia de la ventana consultada.

### Errores


| Código | Cuándo                                                                                                 | `mensaje`                                  |
| ------ | ------------------------------------------------------------------------------------------------------ | ------------------------------------------ |
| `400`  | Body vacío, JSON inválido, `paginaId` faltante, `rango` inválido, fecha mal formada, `hasta` < `desde` | Detalle del problema                       |
| `404`  | La página (`paginaId`) no existe en `PancakePaginas`                                                   | `Página no encontrada: '...'`              |
| `405`  | Método distinto de POST                                                                                | `Método no permitido. Use POST.`           |
| `500`  | Error inesperado.                                                                                      | `Error interno al consultar estadísticas.` |


```json
{ "error": true, "mensaje": "rango inválido: 'semana'. Valores válidos: 'hoy', 'ayer', 'personalizado'." }
```

---

## Flujo interno

```
FunctionHandler (Function.cs)  [API Gateway]
  -> valida método POST y body
  -> deserializa EntradaConsulta
  -> ConsultarEstadisticasUseCase.EjecutarAsync(entrada, ahora)
       -> valida paginaId
       -> CalculadorVentana.Calcular(rango, desde, hasta, ahora)   [since/until en -05:00]
       -> IPaginaTokenRepository.ObtenerAsync(paginaId)             [Mongo: PancakePaginas + PancakeCuentasPrincipales]
            -> null -> PaginaNoEncontradaException (404)
       -> IEstadisticasPancakeClient.ObtenerCampaniasAsync(paginaId, tokenPagina, since, until)
            -> si el token está vencido (success:false o ausente):
                 -> GenerarTokenPaginaAsync(paginaId, tokenUsuario)  [POST generate_page_access_token]
                 -> reintenta ObtenerCampaniasAsync con el token fresco
  -> RESPONSE JSON (NO persiste nada)
```

---

## Arquitectura Clean Architecture

```
ApiLambdaConsultarEstadisticasPagina/
├── Function.cs                              ← Composition Root + mapeo de errores a HTTP
├── Dominio/
│   ├── Entidades/ Campaña.cs · VentanaConsulta.cs
│   └── Excepciones/ PaginaNoEncontradaException.cs
├── Aplicacion/
│   ├── CasosUso/ ConsultarEstadisticasUseCase.cs · CalculadorVentana.cs
│   ├── DTO/ EntradaConsulta.cs · SalidaConsulta.cs
│   └── Interfaces/
│       ├── Repositorios/ IPaginaTokenRepository.cs
│       └── Servicios/    IEstadisticasPancakeClient.cs
└── Infraestructura/
    ├── Repositorio/ PaginaTokenRepository.cs · PaginaTokenDocuments.cs
    └── Servicios/ EstadisticasPancakeClient.cs · Pancake/(PancakeStatsDtos, PancakeApiException)
```

El `UseCase` depende de **interfaces** (`IPaginaTokenRepository`, `IEstadisticasPancakeClient`), no de las implementaciones concretas — por eso se testea con dobles (fakes). El `Function` es el **Composition Root**: el único lugar donde se instancian las concretas.

---

## Variables de entorno


| Variable          | Descripción                                     | Valor ejemplo  |
| ----------------- | ----------------------------------------------- | -------------- |
| `CADENA_CONEXION` | Cadena MongoDB/DocumentDB (cifrada AES-256-ECB) | String cifrado |
| `DATABASE_NAME`   | Base de datos                                   | `"LogighoDB"`  |


Solo **lee** Mongo (para el token). **No** usa `CADENA_CONEXION_SQL` — no persiste.

---

## Configuración Lambda


| Parámetro    | Valor                                                                                                  |
| ------------ | ------------------------------------------------------------------------------------------------------ |
| Runtime      | `dotnet8`                                                                                              |
| Handler      | `ApiLambdaConsultarEstadisticasPagina::ApiLambdaConsultarEstadisticasPagina.Function::FunctionHandler` |
| Memory       | `512 MB`                                                                                               |
| Timeout      | `30 segundos`                                                                                          |
| Architecture | `x86_64`                                                                                               |
| VPC          | Sí (subnets/SG — DocumentDB + salida a internet a Pancake)                                             |


---

## Dependencias externas


| Servicio             | Uso                                                                    |
| -------------------- | ---------------------------------------------------------------------- |
| `Pancake (pages.fm)` | `GET /statistics/pages_campaigns` y `POST /generate_page_access_token` |
| `API Gateway`        | Expone el endpoint y valida el JWT (authorizer)                        |


---

## Historial de cambios


| Fecha      | Autor        | Cambio                                                                                                                                                       |
| ---------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 2026-07-27 | Iker Acevedo | Creación. Endpoint on-demand read-only; rango hoy/ayer/personalizado; fallback de token de página (genera fresco si vencido); Clean Architecture + 13 tests. |


---

## Observaciones

- **Read-only:** a diferencia del pipeline, no escribe en Mongo ni en la RDS. Solo lee el token y devuelve el resultado.
- **Fallback de token:** si el `tokenAccesoPagina` guardado está vencido, se genera uno fresco con el token de usuario de la cuenta madre (`PancakeCuentasPrincipales`) y se reintenta una vez.
- **Campo `paginaId`:** el documento en `PancakePaginas` usa el campo `paginaId` (no `pageId`) — el modelo de lectura lo mapea con `[BsonElement("paginaId")]`. Confundirlo devuelve 404.
- **Ventana en Colombia:** misma matemática -05:00 del pipeline; `ayer` termina en `23:59:59` (no en `00:00` del día siguiente) para no arrastrar datos del otro día.
- Pendiente: comparar `since`/`until` contra los que usa Pancake internamente por si hay que ajustar el formato.

