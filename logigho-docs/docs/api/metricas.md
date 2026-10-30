# API de Métricas (entrega por ciudad)

Base: `/metricas/entrega-ciudad`. Las rutas tienen `authorization=NONE` en el gateway: **la lambda** verifica la firma del JWT y luego busca el `sub` en `Users` ([Seguridad](../backend/metricas/seguridad.md)). Sin token → **401** (`SIN_TOKEN`); rol sin permiso → **403**.

| Entorno | URL base |
|---|---|
| Producción | `https://7elgyten88.execute-api.us-east-1.amazonaws.com/Produccion` |
| PreProd | `https://8u5isw5u30.execute-api.us-east-1.amazonaws.com/PreProd` |

**Encabezados:** `Token` = `id_token` de la sesión (también se acepta `Authorization: Bearer …`) y `headersecurity` (como el resto del sitio). CORS abierto (`*`), con `OPTIONS` por ruta.

## Endpoints

| Método | Ruta | Rol | Respuesta |
|---|---|---|---|
| GET | `/metricas/entrega-ciudad?desde=yyyy-MM&hasta=yyyy-MM` | Todos (autenticado) | 200 filas por ciudad con conteos por transportadora + `fechaCorte` + `minimoGuiasPorDefecto` + `ajusteVisual`. 400 si el rango es inválido |
| GET | `/metricas/entrega-ciudad/corridas?limite=20` | CEO, Desarrollador | 200 lista de corridas, de la más nueva a la más vieja (`limite` 1-100) |
| POST | `/metricas/entrega-ciudad/corridas` · body `{ "alcance": "TODO" }` | CEO, Desarrollador | 202 `{ corridaId, estado }`; 409 si hay una `EN_CURSO`; 400 alcance inválido |
| GET | `/metricas/entrega-ciudad/corridas/{id}` | CEO, Desarrollador | 200 detalle de la corrida (con auditoría); 404 si no existe |
| GET | `/metricas/entrega-ciudad/configuracion` | CEO, Desarrollador | 200 configuración |
| PUT | `/metricas/entrega-ciudad/configuracion` | CEO, Desarrollador | 200 `{ mensaje }`; 400 con `errores[]` si es inválida |

Cualquier otra ruta → 404; método no permitido → 405.

## Envoltorio y errores

Todas las respuestas usan el envoltorio `RespuestaGeneral` del sitio. La lambda serializa con Newtonsoft **sin política camelCase**, así que las claves del envoltorio y de los DTO llegan en **PascalCase** (`Error`, `Mensaje`, `Resultado`, `FechaCorte`, `Filas`…). El front lee ambas grafías.

```json
{ "Error": true, "Mensaje": "SIN_TOKEN", "Resultado": null }
```

| Código | `Mensaje` | Cuándo |
|---|---|---|
| 401 | `SIN_TOKEN` | Sin token, token con firma inválida, vencido o de otro emisor |
| 401 | `USUARIO_NO_ENCONTRADO` | Token válido pero el `sub` no existe en `Users` (al ver el resumen) |
| 403 | `SIN_ROL_PERMITIDO` | Usuario sin rol `CEO` ni `Desarrollador` en una ruta de administración |
| 400 | `DESDE_INVALIDO`, `HASTA_INVALIDO`, `RANGO_INVALIDO` | Rango de meses inválido |
| 400 | `ALCANCE_INVALIDO` | Alcance distinto de `TODO`, `ULTIMOS_6`, `ULTIMOS_3` |
| 400 | `CONFIGURACION_INVALIDA` | PUT inválido; el detalle viene en `Resultado.errores[]` |
| 404 | `CORRIDA_NO_ENCONTRADA` | Id de corrida inexistente |
| 409 | `CORRIDA_EN_CURSO` | Ya hay una corrida viva (menos de 20 min) |
| 500 | `Error interno.` | Con `Resultado.traceId` para buscar en los logs |

## Respuesta del resumen (`GET /metricas/entrega-ciudad`)

```jsonc
{
  "fechaCorte": "2026-10-30T12:49:15Z",   // fin de la última corrida FINALIZADA; null si no hay
  "corridaId": "561a65ae-...",
  "minimoGuiasPorDefecto": 30,
  "ajusteVisual": { "activo": false, "porcentajeMinimo": 60, "modo": "POR_CIUDAD" },
  "transportadoras": ["D2E", "ENVIA", "INTERRAPIDISIMO", "SERVIENTREGA"],
  "filas": [
    {
      "mun8": "11001000",
      "ciudad": "BOGOTA",
      "departamento": "BOGOTÁ, D.C.",
      "porTransportadora": {
        "INTERRAPIDISIMO": { "entregas": 55, "devoluciones": 5, "finales": 60, "porcentaje": 91.7 },
        "ENVIA":           { "entregas": 40, "devoluciones": 20, "finales": 60, "porcentaje": 66.7 }
      }
    }
  ]
}
```

- `porcentaje` es `null` cuando `finales = 0`.
- **Los porcentajes son siempre reales.** `ajusteVisual` solo informa los parámetros con los que el **front** ajusta lo que muestra ([Ajuste visual](../backend/metricas/ajuste-visual.md)).
- Sin corridas finalizadas: 200 con `filas: []` y `fechaCorte: null`.

## Corrida (`GET /corridas`, `GET /corridas/{id}`)

Campos base: `corridaId`, `origen` (`PROGRAMADA`|`MANUAL`), `alcance` (`TODO`|`ULTIMOS_6`|`ULTIMOS_3`), `mesDesde`, `estado` (`EN_CURSO`|`FINALIZADA`|`FALLIDA`|`OMITIDA`), `solicitadoPor` (el `sub` o `scheduler`), `solicitadoPorNombre` (nombre para mostrar: el `username` de `Users`, "Programador (automático)" para el Scheduler; null si el usuario no existe), `inicio`, `fin`, `duracionSeg`, `guiasLeidas`, `guiasEntrega`, `guiasDevolucion`, `noCuentaPorMotivo` (mapa motivo → número), `porcentajeCiudadResuelta`, `documentosEscritos`, `documentosBorrados`, `error`.

Solo en el **detalle** (`GET /corridas/{id}`): `topSinCiudad` (lista `{ ciudad, departamento, guias }`, hasta 50) y `estadosObservados` (lista `{ transportadora, estado, guias, clase }`). El panel de administración usa `estadosObservados` para listar **todos los estados disponibles** al clasificarlos.

Motivos de `noCuentaPorMotivo`: `TRANSITO_U_OTRO`, `POR_FECHA_INDEFINIDA`, `SIN_CIUDAD`, `TRANSPORTADORA_EXCLUIDA`, `TRANSPORTADORA_DESCONOCIDA`, `SIN_FECHA_CARGA`.

Una corrida `EN_CURSO` con más de 20 minutos de antigüedad se considera **caída**: el candado se libera solo y el panel la muestra como "CAÍDA".

## Configuración (`GET`/`PUT /configuracion`)

`version`, `alcanceProgramado`, `minimoGuiasPorDefecto`, `estadosEntrega[]`, `estadosDevolucion[]`, `estadosPorFecha[]`, `mapaTransportadoras` (texto → código), `transportadorasExcluidas[]`, `aliasCiudades` (lista `{ ciudad, departamento, mun8 }`), `ajusteVisual` (`{ activo, porcentajeMinimo, modo }`), `actualizadoPor` (id de Cognito, auditoría), `actualizadoPorNombre` (nombre para mostrar, resuelto contra `Users`; solo lectura, no se guarda), `fechaActualizacion`.

El PUT rechaza con 400: un estado presente en dos listas, un alcance inválido, un mínimo de guías < 1, una clave de mapa vacía, un alias con `mun8` inexistente, un `porcentajeMinimo` del ajuste fuera de 1-99 o un `modo` inválido. Al guardar, `version` sube en 1 y se registran `actualizadoPor` (el `sub`) y la fecha. Los cambios aplican desde la **próxima corrida**.

## Ejemplos

Sin token (humo del pipeline; debe dar 401):

```bash
curl -s https://7elgyten88.execute-api.us-east-1.amazonaws.com/Produccion/metricas/entrega-ciudad
```

Lanzar una corrida (cuerpo y encabezados):

```http
POST /Produccion/metricas/entrega-ciudad/corridas
Token: <id_token>
headersecurity: <valor del front>
Content-Type: application/json

{ "alcance": "TODO" }
```

Respuesta `202`: `{ "Error": false, "Mensaje": null, "Resultado": { "corridaId": "…", "estado": "EN_CURSO" } }`.

La colección de Postman con todos los casos (401, 403, 400, 404, 405, 409, 202, 200 y la firma alterada) está en `specs/004_resumen_entrega_ciudad/pruebas/` del workspace.
