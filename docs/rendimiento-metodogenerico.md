# Rendimiento — metodoGenerico / CargarGuias

Registro de cambios de la iniciativa de rendimiento, por qué se hicieron y qué métrica los valida.
Rama: `fix/rendimiento-metodogenerico`.

## Cambio 1 — Filtro numérico exacto sin regex (DocumentRepository.cs)

**Fecha:** 2026-09-09
**Archivos:** `ApiLambdaCrudGenericoAOT/Infraestructura/Repositorio/DocumentRepository.cs`,
`ApiLambdaCrudGenericoAOT/Infraestructura/Repositorio/FiltroGenericoHelper.cs` (nuevo)

### Problema

Cuando un parámetro de filtro llega como texto pero representa un número (ej. `Numeropreenvio=123456789`,
usado por la Búsqueda Histórica en `pedidos.component.ts`), el código armaba:

```csharp
filterBuilder.Or(
    filterBuilder.Eq(p.Key, numeroLong),
    filterBuilder.Regex(p.Key, new BsonRegularExpression(strNumLong, "i"))
);
```

Un `$regex` sin anclar **nunca puede usar un índice** — MongoDB/DocumentDB tiene que recorrer la
colección completa documento por documento para evaluarlo, aunque el campo tenga un índice
(`Numeropreenvio_1` en `PedidosInter`, confirmado en el dump de índices). Esto es la causa raíz
del reporte de usuarios de "la búsqueda histórica de pedidos es lenta y a veces no encuentra el dato".

### Por qué existía el regex

El campo puede estar guardado como número en unos documentos y como string en otros
(inconsistencia real de esquema en la colección). El regex era una forma de cubrir ambos casos,
a costa de perder el índice.

### Cambio

Como el valor que llega es una coincidencia **exacta** (el usuario escribió el número completo, no
una búsqueda parcial), se reemplaza el regex por un segundo `Eq` contra la representación en texto:

```csharp
filterBuilder.Or(
    filterBuilder.Eq(campo, valorNumerico),
    filterBuilder.Eq(campo, valorComoTexto)
);
```

Ambas cláusulas del `$or` son comparaciones exactas indexables — MongoDB puede resolver cada una
usando el índice del campo, sin escanear la colección.

Extraído a `FiltroGenericoHelper.FiltroIgualdadNumericaOTexto` (antes estaba repetido igual en 3
lugares de `DocumentRepository.cs`: búsqueda paralela anidada, búsqueda paralela principal, y
borrado filtrado) — Single Responsibility + DRY, y permite testear la lógica sin conexión real
a MongoDB.

**No se tocó** el branch de búsqueda parcial de texto (`Regex` sobre campos de texto genuino como
nombre/ciudad) — ahí el regex sigue siendo necesario porque no hay forma exacta de decir "contiene".

### Tests

`ApiLambdaCrudGenericoAOT.Tests/FiltroGenericoHelperTest.cs` — 3 tests nuevos (19/19 total en el
proyecto, todos en verde):
- Filtro exacto no genera `$regex`.
- Matchea el campo guardado como número Y como string, no matchea un valor distinto.
- Preserva el texto original en la cláusula string (ceros a la izquierda, números largos).

### Validación AOT

`dotnet build` compila limpio. `dotnet publish -r linux-x64 --self-contained -p:PublishAot=true`
compila el C# a IL sin errores; el link nativo final falla en Windows con
`Cross-OS native compilation is not supported` — **limitación del entorno de desarrollo (Windows),
no del código**. El link nativo real debe validarse en el pipeline de build (Linux) o WSL con
el SDK de .NET instalado, antes de mergear.

### Métrica que valida el fix (antes/después, en preprod)

| Métrica | Antes | Esperado después |
|---|---|---|
| Duration p99 de `ApiLambdaCrudGenericoAOT` (búsquedas por Numeropreenvio) | hasta 28.7s | cercano al p50 (~8-10ms) |
| Invocaciones >25000ms en 7 días | 1024 | cercano a 0 para este patrón de búsqueda |
| Resultado "No se encontró el dato" en búsqueda histórica con guía válida | ocurre (timeout antes de completar) | no debería ocurrir para guías existentes |

## Pendiente / próximos cambios de esta iniciativa

- `ReservedConcurrentExecutions` en `ApiLambdaCrudGenericoAOT` (AWS, config) — valor propuesto 150,
  basado en pico real observado de 97 concurrencia simultánea (7-9 sep 2026) × 1.5 de margen.
  Pendiente: valor de `max_connections` del parameter group de DocumentDB para cerrar la cuenta
  contra `MaxConnectionPoolSize` del cliente Mongo.
- `MaxTime` en operaciones `Find` (hoy solo `EjecutarAggregateAsync` lo tiene) — evita que una query
  se quede colgada hasta el timeout duro del Lambda (30s) / API Gateway (29s fijo, no configurable).
- Índices compuestos filtro+orden (Fase 03 del plan original) — para los casos que sí necesitan
  ordenar por un campo distinto de `_id`.
- Cache de catálogos en frontend + retry con backoff en 429 (throttling del Lambda).
