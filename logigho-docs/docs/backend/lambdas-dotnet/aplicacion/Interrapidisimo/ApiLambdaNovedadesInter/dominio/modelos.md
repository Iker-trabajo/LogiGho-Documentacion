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
    "Estado": "Respondida",              // Enviada | Respondida | Fallida | NoEncontrada
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
  "UltimaRespuesta": { … },              // respuesta completa de ws-inter, se sobrescribe
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
