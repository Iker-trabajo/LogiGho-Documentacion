## Autor: Iker Acevedo
Fecha creación: 2026-09-16

Estado: preprod

# Orquestador — `LiquidarPedidosDinamicoUseCase`

Es la única clase del motor Dinámico que toca infraestructura (Mongo). Todo lo que invoca — Clasificador, Validador, Resolvedor, Calculadoras, `ConstructorPeticion` — es dominio puro, sin I/O. Vive en `Aplicacion/CasosUso/Liquidacion/LiquidarPedidosDinamicoUseCase.cs`, junto al `DepurarLiquidacionUseCase.cs` legacy (mismo directorio, distinto namespace de C#).

---

## Partición: cómo decide qué motor usa cada pedido

```mermaid
--8<-- "backend/lambdas-dotnet/lambdas/ApiLambdaLiquidacionesLogighoAOT/diagramas/orquestador-particion.mmd"
```

> Fuente: **[carpeta de diagramas](../diagramas/README.md)**.

`Particionar(JArray pedidos, List<ConfiguracionTransportadora> configuraciones)` agrupa las configuraciones de `ConfiguracionLiquidacionTransportadora` por `NombreEnPedidos` (normalizado: trim + mayúsculas), y por cada pedido compara su campo `Transportadora` (normalizado igual) contra ese diccionario:

- Existe configuración **y** `MotorLiquidacion` es `Dinamica` o `Sombra` → lote **Dinámicos**.
- Cualquier otro caso (`Legacy`, `Deshabilitado`, o sin configuración) → lote **Legacy**.

Devuelve `(JArray Dinamicos, JArray Legacy)`. Los dos lotes se procesan en la misma invocación de `Function.cs` — no son excluyentes.

---

## `LiquidarAsync` — el pipeline sobre el lote Dinámico

Por cada pedido del lote `Dinamicos`:

1. Mapea el pedido crudo a `PedidoLiquidable` (`MapeadorLiquidacionDinamica.MapearPedido`).
2. Corre pipeline completo (Clasificar → Validar trayecto → Resolver tarifa → Calculadoras → `ConstructorPeticion`) — ver [Visión general § Pipeline](overview.md#piezas-en-orden-de-uso).
3. Si algún paso no puede avanzar, la guía queda registrada como `GuiaNoLiquidada` con su `MotivoNoLiquidacion` — sin reintento automático.
4. Inserta los documentos resueltos directo en Mongo, vía `IDocumentRepository.InsertarLiquidacionesLogighoAsync` — la **misma** colección y el **mismo** repositorio que usa Legacy en su rama de pruebas locales (`#if DEBUG`). Ver [Infraestructura](infraestructura.md) para la confirmación completa.
5. Devuelve `ResumenLiquidacionDinamica(DocumentosLiquidados, NoLiquidadas)`.

Una transportadora en `Sombra` corre exactamente este mismo pipeline — la única diferencia es que el resultado **no se persiste**. Sirve para validar una configuración nueva (tarifas, trayectos, tope de kilos) contra pedidos reales, sin arriesgar un solo documento en `LiquidacionesLogigho`.

---

## `Function.cs` — el entry point real

Es quien de verdad arranca la Lambda:

1. Obtiene de Mongo `ConfiguracionLiquidacionTransportadora`, `TarifasTransportadoraTienda` y `TrayectosTransportadora`.
2. Los mapea a dominio con `MapeadorLiquidacionDinamica`.
3. Llama `Particionar`.
4. Ejecuta `LiquidarAsync` sobre el lote Dinámico.
5. Si quedan pedidos en el lote Legacy, invoca `DepurarLiquidacionUseCase.DepuraLiquidacionInter(...)` sobre ellos.

---

## Fases del proyecto

El `README.md` de `LiquidacionDinamica/` no numera fases F1-F4 de forma explícita y centralizada — se mencionan sueltas en comentarios de código (`TarifaTransportadoraTienda.cs`, `TrayectoTarifa.cs`). Reconstruido a partir de esos comentarios y del estado real del código:

| Fase | Qué cubre | Estado |
|---|---|---|
| F1 | Modelos de configuración + validación externa (script + front) | Hecho |
| F2 | Calculadoras de peso, dominio puro | Hecho |
| F3/F4 | Infraestructura y capa de aplicación completas — orquestación end-to-end, persistencia de auditoría, ejecución real de pagos de Marketing/Referido | **Parcial** — `LiquidarPedidosDinamicoUseCase` y `MapeadorLiquidacionDinamica` ya cubren buena parte de esto, pero la colección `AuditoriaLiquidaciones` todavía no tiene quién escriba en ella (ver [Frontend → Auditoría de Excepciones](../../../../../frontend/views/director-de-operaciones/costos-transportadora/components/auditoria-excepciones.md)) |

> El README puede estar desactualizado respecto al código — esto es una foto tomada el 2026-09-16, no una fuente viva.

---

## Tests

No hay un archivo de test propio para `LiquidarPedidosDinamicoUseCase` dentro de `ApiLambdaLiquidacionesLogighoAOT.Tests/LiquidacionDinamica/` — la carpeta cubre solo el dominio puro (Calculadoras, Políticas de Peso, Validación) y el mapeador de infraestructura. El propio código del orquestador señala que ese "cableado" (leer Mongo, particionar, invocar Legacy) no tiene cobertura de test dedicada todavía.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-09-16 | Iker Acevedo | Documentación inicial del orquestador y de `Function.cs`. |
