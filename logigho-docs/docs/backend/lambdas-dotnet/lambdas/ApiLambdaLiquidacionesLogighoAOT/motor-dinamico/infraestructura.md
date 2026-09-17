## Autor: Iker Acevedo
Fecha creación: 2026-09-16

Estado: preprod

# Infraestructura — Mapeo y persistencia

El motor Dinámico tiene una sola pieza de infraestructura propia: `LiquidacionDinamica/Infraestructura/Mapeo/MapeadorLiquidacionDinamica.cs`. No tiene un repositorio propio — reutiliza `IDocumentRepository`/`DocumentRepository`, el mismo que ya usa el motor Legacy.

---

## `MapeadorLiquidacionDinamica`

Convierte los `JArray`/`JToken` crudos que vienen de Mongo a los modelos de dominio del motor Dinámico:

| Método | De | A |
|---|---|---|
| `MapearConfiguraciones` | `ConfiguracionLiquidacionTransportadora` | `List<ConfiguracionTransportadora>` |
| `MapearTarifas` | `TarifasTransportadoraTienda` | `List<TarifaTransportadoraTienda>` |
| `MapearTrayectos` | `TrayectosTransportadora` | `List<TrayectoTransportadora>` |
| `MapearPedido` | `PedidosInter` (nombres de campo legacy, con espacios: `"Fecha Entrega"`, etc.) | `PedidoLiquidable`, o `null` si no hay `Numeropreenvio` válido |
| `MapearCostosOperativos` | `AdministracionCostos` | `CostosOperativosTienda`, con fallback a todo-cero si la tienda no tiene costos configurados |

Las tres primeras colecciones (`ConfiguracionLiquidacionTransportadora`, `TarifasTransportadoraTienda`, `TrayectosTransportadora`) son **nuevas** — sus campos ya están en PascalCase idéntico al modelo de dominio, así que la deserialización con Newtonsoft es directa. `MapearPedido` y `MapearCostosOperativos` sí necesitan traducir nombres, porque leen colecciones legacy (`PedidosInter`, `AdministracionCostos`) que no se tocaron.

---

## Por qué escribe en `LiquidacionesLogigho` (no colección aparte)

**Decisión explícita, no descuido:**
- V1 del diseño contempló colección "sandbox" separada (protegía pero añadía complejidad).
- Se descartó: pre-prod ya es entorno aislado (Mongo ≠ producción).
- Aislar **además** dentro pre-prod = diferencia comportamiento que desaparece en producción.
- Replicar comportamiento desde ya (misma colección, mismo formato) = permite comparar liquidaciones reales Motor Dinámico vs Legacy sin traducción.

Ambos motores usan **idéntico repositorio**:
- `LiquidarPedidosDinamicoUseCase.LiquidarAsync` → `_documentRepository.InsertarLiquidacionesLogighoAsync(bson)`.
- `DocumentRepository.cs:392-401` → `GetCollection<BsonDocument>("LiquidacionesLogigho")` + `InsertManyAsync`.
- **Mismo** método, **misma** colección que Legacy en `#if DEBUG` (línea ~462 `DepurarLiquidacionUseCase.cs`).
- Comentario original Legacy lo deja claro: intencionado, pruebas = producción.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-09-16 | Iker Acevedo | Documentación inicial del mapeador y confirmación de que ambos motores escriben en la misma colección. |
| 2026-09-16 | Iker Acevedo | Se agregaron 3 tests de regresión (`MapeadorLiquidacionDinamicaTests.cs`) que deserializan JSON con forma real de Mongo — el bug de `ReadOnlyCollection` (ver [Modelos](modelos.md)) era invisible para la suite anterior porque ningún test pasaba por `JsonConvert.DeserializeObject` de verdad. |
