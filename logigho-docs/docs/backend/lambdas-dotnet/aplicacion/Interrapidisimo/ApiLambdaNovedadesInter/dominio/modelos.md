## Autor: Iker Acevedo

Fecha creacion: 2026-10-18  
Estado: desarrollo

---

# Modelos de dominio y datos

## `PedidosInter.MotivoNovedadInter` (campo nuevo)

Único campo que esta lambda escribe en `PedidosInter` (con `$set`, nunca reemplazando el documento).

```
"REHUSADO / SE NEGO A RECIBIR NOTIFICACION - https://www.google.com/maps?q=10.868765635,-74.78554886"
```

- Es texto para mostrar; **no se parsea**. Sin geo queda solo el motivo.
- Se conserva cuando la guía sale de novedad (es lo último que reportó Inter).
- La colección `PedidosInter` mezcla transportadoras y tiene más de 100 campos planos; por eso el detalle no va ahí.
- `Numeropreenvio` es `NumberLong`; ws-inter usa la guía como texto.

## Colección `SeguimientoNovedadesInter`

Un documento por guía. El nombre evita confundirse con `NovedadesLogigho`, que son novedades **financieras**.

```jsonc
{
  "Guia": "240063616541",                // texto, como lo usa ws-inter
  "Numeropreenvio": 240063616541,        // NumberLong: clave de búsqueda (índice único)
  "PedidoId": ObjectId("…"), "Idtienda": "…", "Tienda": "…",
  "EstadoSeguimiento": "Activo",         // Activo | Cerrado (salió de novedad: no se consulta más)
  "Prioridad": "Alta",                   // Alta | Normal
  "HuellaPedido": "Intento de entrega|No logramos hacer la entrega",   // Estado|Descripcion Asociada al solicitar
  "Solicitud": {
    "Estado": "Respondida",              // Enviada | Respondida | Fallida (sin callback o error del proveedor) | NoEncontrada (encontrado:false)
    "Ref": "…", "JobId": "…", "FechaEnvio": ISODate, "FechaRespuesta": ISODate,
    "Intentos": 0,                       // envíos seguidos sin respuesta
    "UltimoError": null
  },
  "UltimoMotivo": {
    "Motivo": "REHUSADO / SE NEGO A RECIBIR NOTIFICACION",
    "EstadoInter": "En Proceso de Devolución",
    "Fecha": ISODate("2026-10-17T17:18:59Z"),       // UTC
    "FechaRaw": "17/10/2026 12:18:59 p. m.",        // tal como la muestra Inter
    "Ciudad": "MALAMBO\\ATLA\\COL", "Mensajero": "…", "Observacion": "Descargado desde APP",
    "Geo": { "Latitud": 10.868765635, "Longitud": -74.78554886, "Url": "https://www.google.com/maps?q=…" }
  },
  "HuellaMotivos": "a3f1…",              // hash de TODOS los motivos: si no cambia, el motivo no cambió
  "Historial": [ { "Motivo": { … }, "Huella": "…", "DetectadoEn": ISODate } ],   // solo cambios, tope 50
  "UltimaConsultaEn": ISODate,           // consultado_en de ws-inter: ignora callbacks viejos
  "UltimaRespuesta": { … },              // solo guia, consultado_en, encontrado, estado_actual, ultimo_motivo, flujo, motivos (+ job_id, ref, status); sin datos personales; se sobrescribe
  "ConsultasTotales": 3, "ConsultasSinCambio": 2,
  "FechaCreacion": ISODate, "FechaActualizacion": ISODate, "FechaCierre": null
}
```

Todas las fechas son BSON `Date` en UTC.

### Índices (comandos en el [runbook](../infraestructura/runbook.md), sección A2)

| Índice | Para qué |
| ------ | -------- |
| `{ Numeropreenvio: 1 }` único | Una guía = un seguimiento; evita duplicados por envíos simultáneos |
| `{ "Solicitud.Ref": 1 }` | Encontrar el seguimiento por la referencia que devuelve ws-inter |
| `{ EstadoSeguimiento: 1 }` | Cerrar los seguimientos activos que salieron de novedad |

### Escrituras

Todas son parciales (`$set`, `$inc`, `$push`, `$setOnInsert`): el envío (Step) y el callback (webhook) pueden tocar el
mismo seguimiento casi a la vez sin pisarse. Nunca se reemplaza el documento.

## Clases (`Dominio/`)

| Clase | Responsabilidad |
| ----- | --------------- |
| `ReglaSeleccionNovedad` | ¿La guía está en novedad? ¿Qué prioridad tiene? |
| `PlanificadorEnvios` | Decide Enviar / Saltar / MarcarFallida (tabla del [caso de uso](../casos-uso/solicitar-novedades-usecase.md)) |
| `ExtractorMotivoNovedad` | Del JSON de ws-inter saca motivo, geo, huella y texto |
| `SeguimientoNovedad` | Documento de la colección anterior |
| `PedidoInterNovedad` | Los 6 campos de `PedidosInter` que se leen |

---

## Campos nuevos de `SeguimientoNovedadesInter` (historia 027, spec 012)

Los escribe **solo** la función [Distancias](../ApiLambdaNovedadesInterDistancias.md), con `$set` parcial de estos cuatro campos
(nunca reescribe el documento: el callback de ws-inter puede estar tocando el mismo seguimiento). Reglas en
[Reglas de la distancia](reglas-distancia.md). Observado: `Dominio/Entidades/Distancias.cs`, `DistanciasRepositorio.cs`.

```jsonc
{
  "DireccionPedido": {                      // geocodificación de la dirección del pedido (una por guía)
    "HuellaDireccion": "sha256…",           // regla + dirección + ciudad + departamento normalizados
    "Latitud": 4.6, "Longitud": -74.08,     // ejemplo inventado
    "TipoUbicacion": "ROOFTOP",             // ROOFTOP | RANGE_INTERPOLATED | GEOMETRIC_CENTER | APPROXIMATE
    "CoincidenciaParcial": false,
    "Localidades": ["Bogotá"],              // opcional: locality / administrative_area_level_2 del resultado
    "ConfirmacionDireccion": "Confirmada",  // opcional: Confirmada | NoConfirmada | ausente
    "Estado": "Encontrada",                 // Encontrada | NoEncontrada | Error
    "GeocodificadaEn": ISODate
  },
  "Distancias": [                           // una por intento; tope de 50, igual que Historial
    {
      "HuellaGeo": "4.610000,-74.070000",   // "lat,lng" a 6 decimales: llave de la caché de rutas
      "FechaIntento": ISODate,
      "LatitudNovedad": 4.61, "LongitudNovedad": -74.07,
      "Metros": 640, "MetrosLineaRecta": 410, "Modo": "DRIVE",
      "Color": "rojo",                      // verde | amarillo | rojo | null
      "Estado": "Calculada",                // Calculada | SinGeo | DireccionNoEncontrada | BajaConfianza | Error
      "UrlRuta": "https://www.google.com/maps/dir/?api=1&origin=…&destination=…&travelmode=driving",
      "CalculadaEn": ISODate
    }
  ],
  "DistanciaUltima": { /* copia de la entrada que corresponde a UltimoMotivo */ },
  "HuellaDistancias": "<HuellaMotivos>|v3"  // versión de la regla con la que se calculó
}
```

| Campo | Significado |
| ----- | ----------- |
| `DireccionPedido` | Resultado de geocodificar `PedidosInter.Direccion + Ciudad`. Es dato personal derivado: no se loguea ni sale en los Excel. Solo se repite si cambia `HuellaDireccion` |
| `DireccionPedido.Localidades` | Opcional (`BsonIgnoreIfNull`). Con ellas se valida que la ciudad del resultado sea la del pedido |
| `DireccionPedido.ConfirmacionDireccion` | Opcional. Resultado de la segunda geocodificación, solo para guías con ruta roja. No se repite mientras la dirección no cambie |
| `Distancias[]` | Un resultado por intento (cada entrada de `Historial` más el último motivo). Sin coordenada: `Estado = SinGeo` |
| `DistanciaUltima` | Copia de la distancia del intento que corresponde a `UltimoMotivo`; es lo que lee la vista (columna) y el reporte 1 |
| `HuellaDistancias` | `"{HuellaMotivos}\|v3"`. Si es distinta de esa expresión, la guía es **pendiente** y se recalcula. Si no se pudo completar (Google falló, falta el pedido) no se fija y se reintenta |

`origin` del enlace es la dirección del pedido; `destination`, el punto del mensajero. Valores con `InvariantCulture` (punto decimal).

## Colección `ReportesNovedadesInter` (nueva)

La escribe solo la función [Reportes](../ApiLambdaNovedadesInterReportes.md): un documento por corrida. De la última `Exitosa` sale la frontera de «Nuevas».

```jsonc
{
  "_id": ObjectId("…"),
  "FechaInicio": ISODate, "FechaFin": ISODate,
  "Origen": "programado",                 // programado | slack
  "UsuarioSlack": "U0000EJEMPLO",         // solo si Origen = slack (id inventado)
  "Estado": "Exitosa",                    // Exitosa | Fallida
  "FilasReporte1": { "Nuevas": 3, "Acumulado": 40 },
  "FilasReporte2": { "Nuevas": 7, "Acumulado": 120 },
  "Error": null                           // causa corta si Fallida
}
```

## Colección `CandadosNovedadesInter` (nueva)

Candado entre corridas de Distancias: `{ "_id": "distancias", "Dueno": "<id de la corrida>", "ExpiraEn": ISODate }`. Se toma con un `upsert` atómico y se libera
al terminar; vence a los 15 min por si la función muere sin liberarlo.

## Índices nuevos (mejora M-256)

Los crea Iker en prod (en PreProd puede la IA). Comandos y orden en el [runbook](../infraestructura/runbook.md#d-historia-027-distancias-y-reportes).

| Colección | Índice | Para qué |
| --------- | ------ | -------- |
| `CandadosNovedadesInter` | `{ ExpiraEn: 1 }` con `expireAfterSeconds: 0` | TTL: limpia candados vencidos (el código ya comprueba el vencimiento, así que funciona aunque tarde) |
| `ReportesNovedadesInter` | `{ Estado: 1, FechaInicio: -1 }` | Hallar la última corrida `Exitosa` |
| `SeguimientoNovedadesInter` | `{ HuellaMotivos: 1 }` | **Condicional:** solo si el `explain` lo pide. Hoy la selección de pendientes lee una proyección mínima y compara en memoria |

## Clases nuevas (`Dominio/`)

| Clase | Responsabilidad |
| ----- | --------------- |
| `ReglaColorDistancia` | Metros → verde / amarillo / rojo (250 y 500) |
| `ReglaConfianzaGeocodificacion` | `ROOFTOP`/`RANGE_INTERPOLATED` + ciudad igual a la del pedido |
| `ReglaConfirmacionDireccion` | Segunda geocodificación a ≤ 150 m |
| `ValidadorCoordenada`, `Haversine` | Defensa de coordenadas y línea recta de referencia |
| `SanitizadorDireccion`, `VersionReglaGeocodificacion`, `TextoGeografico` | Limpieza de la dirección, versión de la regla (`v3`), huellas y enlaces |
| `NormalizadorTexto`, `ListaMotivosExcluidos` | Clave de comparación de motivos y los 8 excluidos |
| `FiltroReporte1`, `FiltroReporte2` | Qué guías entran a cada reporte |
| `DireccionPedido`, `DistanciaIntento`, `CorridaReporte` | Entidades de las secciones anteriores |
