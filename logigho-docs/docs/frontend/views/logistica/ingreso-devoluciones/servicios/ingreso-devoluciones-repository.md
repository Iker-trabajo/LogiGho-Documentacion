## Autor: Iker Acevedo

Fecha creacion: 2026-08-26

Estado: produccion

## Servicio: IngresoDevolucionesRepository

**Ubicación:** `helpers/ingreso-devoluciones.repository.ts`

---

## ¿Qué hace?

Único punto del módulo que habla HTTP. Ningún componente ni el store importa `ConsumoGenericoService`/`DecompressionService`/`GetObjectService` directamente — si mañana cambia el transporte, solo se toca este archivo.

| Método | Endpoint | Uso |
| ------ | -------- | --- |
| `iniciarLote(guias)` | `POST devolucionesMasivo/iniciar` vía `insertarGenerico` | Crea el job |
| `obtenerEstado(jobId, conDetalle?)` | `GET devolucionesMasivo/estado?jobId=X[&detalle=true]` vía `consultarGenerico` | Polling |
| `obtenerAbiertos()` | mismo endpoint, sin `jobId` | Reenganche al entrar al módulo |
| `obtenerHistorial(usuarioEmail)` | `metodoGenerico?coleccion=JobsDevolucion&Usuario=...` | Tabla de historial |
| `obtenerInventarioDeLote(guias)` | `metodoGenerico?coleccion=InventarioDevolucion` + filtro tienda, filtrado en memoria por `guias` | Detalle de aceptadas del lote **activo** (mientras se conoce la lista en memoria) |
| `obtenerInventarioPorJobId(jobId)` | igual, filtrando por `JobId` en la query | Detalle de aceptadas de un lote **histórico** — correlación exacta, ver [ADR-003](../../../../../backend/lambdas-dotnet/aplicacion/Logigho/ApiLambdaDevolucionesMasivo/adr/ADR-003-jobid-en-inventariodevolucion.md) |

---

## `iniciarLote` usa `insertarGenerico`, no `insertarGenericoSinAuditoria`

Decisión explícita de seguir la convención del resto del módulo, aceptando un efecto secundario conocido: como `devolucionesMasivo/iniciar` no tiene el formato `metodoGenerico?coleccion=...`, el `replace()` interno de `insertarGenerico` (que arma el nombre de colección para el registro de auditoría) no encuentra nada que reemplazar. El registro de auditoría queda con `coleccion="devolucionesMasivo/iniciar"` en vez de un nombre de colección real. No afecta el flujo principal — solo ensucia ese log de auditoría puntual.

---

## Filtro por tienda vs. filtro por usuario — dos colecciones, dos criterios

```typescript
// JobsDevolucion: filtra por Usuario — un job pertenece a quien lo subió
obtenerHistorial(usuarioEmail)

// InventarioDevolucion: filtra por TIENDA asignada, no por usuario
obtenerInventarioDeLote(guias)
obtenerInventarioPorJobId(jobId)
```

**Por qué la diferencia es intencional:** una tienda puede tener varios operarios, y una guía de esa tienda la pudo procesar cualquiera de ellos. Filtrar `InventarioDevolucion` por usuario dejaría fuera guías legítimas que procesó un compañero de la misma tienda. `JobsDevolucion` sí filtra por usuario porque un job es propiedad de quien lo creó — mismo criterio de dueño que valida `ConsultarJobUseCase` en el backend.

`tiendasAsignadas()` lee `sessionStorage.getItem('tiendas_asignadas')` — si el usuario tiene `'Todas'`, no aplica ningún filtro (ve todo); si no, arma la lista separada por coma para el parámetro `NombreTienda`.

---

## `obtenerInventarioDeLote` filtra en memoria — por qué

```typescript
// El filtrado por guia se hace en memoria: metodoGenerico no soporta
// "IN" sobre un arreglo grande de guias en la query string.
const buscadas = new Set(guias);
return todas.filter(doc => buscadas.has(doc.Guia));
```

No hay forma de pedirle a `metodoGenerico` "tráeme solo estas 200 guías" — no soporta `IN` sobre un arreglo en la query string. Se trae todo el inventario filtrado por tienda y se recorta en memoria. `obtenerInventarioPorJobId` evita este problema por completo: filtra directo por `JobId` en Mongo, sin traer de más.

---

## Observaciones

- `obtenerInventarioDeLote` (por guías en memoria) sigue existiendo para el flujo del lote **activo**, donde el front todavía tiene la lista de guías enviadas fresca en memoria — no hace falta el `JobId` para ese caso porque no hay ambigüedad de qué lote es.
- Los métodos que consultan `metodoGenerico` dependen de `DecompressionService.decompressGzip` — el backend genérico comprime las respuestas grandes.
