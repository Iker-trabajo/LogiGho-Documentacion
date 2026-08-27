## Autor: Iker Acevedo

Fecha creacion: 2026-08-26

Estado: produccion

## Helper: ingreso-devoluciones.rules.ts

**Ubicación:** `helpers/ingreso-devoluciones.rules.ts`
**Tests:** `ingreso-devoluciones.rules.spec.ts` — funciones puras, sin Angular ni HTTP, 100% testeables sin mocks

---

## ¿Qué hace?

Toda la lógica de negocio del front que no necesita hablar con nada — misma entrada, misma salida siempre. Se separó del resto a propósito: es lo más fácil de testear y lo que menos debería romperse con el tiempo.

---

## Fechas

⚠️ **Ver [ADR-004](../../../../../backend/lambdas-dotnet/aplicacion/Logigho/ApiLambdaDevolucionesMasivo/adr/ADR-004-fechas-utc.md) antes de tocar esto** — el backend guarda las fechas propias del módulo en UTC real, y la conversión a hora Colombia se hace **una sola vez**, acá.

| Función | Qué hace |
| ------- | -------- |
| `parseFecha(valor)` | Normaliza las 2 formas en que puede llegar una fecha: ISO plano (API) o `{ $date }` (Mongo) |
| `aHoraColombia(fechaUtc)` | Resta 5 horas |
| `fechaColombiaDesdeJob(valor)`, `fechaColombiaDesdeInventario(valor)` | `parseFecha` + `aHoraColombia`, con nombres que dejan explícito de qué colección viene el dato |
| `formatFechaLarga(fecha)` | `"25 de Ago del 2026 a las 7:24 pm"` |
| `rangoFechasFiltro(desde, hasta)` | `AAAAMMDD-AAAAMMDD` para el parámetro `fechasFiltro` de `metodoGenerico` |

**`formatFechaLarga` lee con accesores `getUTC*` a propósito, no el pipe `date` de Angular.** El pipe `date` convierte según la zona horaria del **navegador** — la pantalla se vería distinta para un operario en bodega que para alguien mirando desde otro país. La operación es colombiana: la hora mostrada tiene que ser siempre la de Colombia, sin importar dónde esté abierto el navegador. Como la fecha que recibe `formatFechaLarga` **ya** está en hora Colombia (la convirtió `aHoraColombia` antes), aplicarle la zona del navegador la volvería a mover.

⚠️ **Dato viejo:** las filas de `InventarioDevolucion` escritas antes de que el backend migrara a UTC real se van a mostrar 5 horas antes de lo real. Son las guías de las pruebas iniciales del módulo.

---

## Identificador legible del lote

```typescript
derivarIdLote(usuario, fechaInicioColombia)  // "DEV-ACEVEDO0413-260825-192432"
```

Se deriva de `FechaInicio` + `Usuario`, nunca del reloj del navegador ni de un contador guardado — la misma fecha y el mismo usuario dan siempre el mismo identificador, hoy o reconstruido desde el historial dentro de un año. Mismo patrón que usa `relacion-despacho` con `generarIdLote`. El `prefijoUsuario` toma la parte del email antes del `@`, en mayúsculas, sin caracteres que no sean letras o números.

---

## Catálogo de motivos

| Función | Qué hace |
| ------- | -------- |
| `infoMotivo(motivo)` | Traduce a la ficha humana (`CATALOGO_MOTIVOS`). Nunca revienta: si el motivo es `null`/desconocido, devuelve `MOTIVO_DESCONOCIDO` |
| `motivoDesdeTexto(texto)` | Normaliza el campo `MotivoRechazo` de `InventarioDevolucion` — trata `null`, vacío, y el texto literal `"BsonNull"` (bug conocido del backend, ver [modelos](../modelos/ingreso-devoluciones-models.md)) todos igual, como ausencia de motivo |
| `guiaOriginalDe(guia)` | Prefiere `GuiaOriginalExtraida` (la reportó la transportadora); si no existe, cae a `Numeropreenvio` (sale de `PedidosInter`, se llena cuando el pedido sí se encontró — el caso de `PedidoAnulado`/`SinPermisoTienda`). Nunca inventa un número: si no se conoce ninguna de las dos, `null` |
| `guiasReintentables(detalle)` | Las que tienen `MereceReintentoConOtraTransportadora` o motivo `ErrorConsultaExterna` |
| `agruparPorMotivo(detalle)` | `Map<MotivoRechazo \| null, GuiaProcesada[]>` |
| `resumenPorFamilia(detalle)` | Agrupa primero por familia (orden fijo: `dato`, `negocio`, `tecnico`) y dentro de cada familia por motivo puntual — es lo que consume `detalle-lote` para no listar cientos de filas sueltas: el operario lee "180 fallaron porque la transportadora no respondió", no scrollea buscando el patrón a ojo |

---

## Depuración de guías (archivo o escaneo)

```typescript
depurarGuias(crudo: Array<string | number | null | undefined>): GuiasDepuradas
```

Quita vacías, normaliza a texto (para no perder ceros a la izquierda si Excel leyó la guía como número) y quita duplicadas **dentro del mismo envío**, contando cuántas se descartaron de cada tipo. **No** consulta nada contra el backend — la deduplicación contra guías ya procesadas de corridas anteriores la hace el backend (idempotencia).

```typescript
esCandidataGuia(texto)       // /^\d{6,20}$/ — heuristica compartida
sugerirColumnaGuias(muestra) // elige la columna con mas celdas que parecen guia
```

---

## Progreso

```typescript
segundosParaConsiderarseDetenido(fechaUltimaActividad)
```

Replica el mismo umbral (2 minutos) que usa `ConsultarJobUseCase` para decidir `WorkerDetenido` — solo para mostrar una cuenta o explicarle el plazo al operario. **La verdad de si el worker está detenido siempre la dice el campo `WorkerDetenido` de la respuesta**, esta función nunca decide eso por su cuenta.

```typescript
formatDuracion(segundos)                          // "01:05" o "01:01:01"
estimarSegundosRestantes(procesadas, pendientes, segundosTranscurridos)
```

`estimarSegundosRestantes` devuelve `null` cuando todavía no hay información suficiente (0 procesadas o 0 segundos transcurridos) — es mejor no mostrar nada a mostrar `Infinity` o `NaN`, que confunden más de lo que ayudan.

---

## Paginación

```typescript
paginar<T>(filas: T[], pagina: number, tamano: number): Pagina<T>
```

Corta una lista en páginas, **corrigiendo** la página pedida si quedó fuera de rango (en vez de devolver una página vacía) — pasa siempre que el operario está en la página 5 y escribe un filtro que deja 12 resultados. Mostrarle la última página válida es más útil que una tabla en blanco.

---

## Mapeo de documentos crudos

```typescript
mapearLoteHistorial(doc: JobDevolucionDoc): LoteHistorial
mapearDesdeInventario(doc: InventarioDevolucionDoc): GuiaProcesada
```

Aíslan al resto del front del formato crudo de Mongo — si ese formato cambia, solo se toca la función.

**Bug corregido en `mapearLoteHistorial`:** originalmente `fechaFin` leía `doc.FechaInicio` de nuevo por error de copia/pega, así que `fechaFin` siempre quedaba igual a `fechaInicio` para todo lote del historial.

---

## Guías resueltas por ventana de tiempo (histórico previo a `JobId`)

```typescript
resueltasDelLote(docs, desde, hasta): InventarioDevolucionDoc[]
```

⚠️ **Limitación documentada explícitamente en el código:** `InventarioDevolucion` no guarda a qué lote pertenece cada guía para los documentos escritos antes del cambio de [ADR-003](../../../../../backend/lambdas-dotnet/aplicacion/Logigho/ApiLambdaDevolucionesMasivo/adr/ADR-003-jobid-en-inventariodevolucion.md) — la única forma de correlacionarlas es la ventana de tiempo del lote. Si el mismo operario corre dos lotes de la misma tienda solapados, las guías se mezclarían. Se conserva esta función por si algún día hace falta reconstruir el histórico previo al cambio; el flujo actual usa `obtenerInventarioPorJobId` (correlación exacta).
