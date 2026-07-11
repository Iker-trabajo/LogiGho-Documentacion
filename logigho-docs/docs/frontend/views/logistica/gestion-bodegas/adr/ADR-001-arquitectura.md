## ADR-001 — Patrón de arquitectura del módulo

**Autor:** Adalberto González  
**Fecha:** 2026-06-24  
**Estado:** Aceptada

---

### Contexto

Se necesitaba un módulo para gestionar bodegas con operaciones CRUD, asignación de productos y visualización de stock. Se evaluó si construirlo con un componente monolítico o aplicar el mismo patrón en capas ya establecido en `resumen-inventario` y `gestion-devoluciones`.

---

### Opciones consideradas

**Opción A — Componente monolítico:** Todo el HTTP, estado y presentación en `GestionBodegasComponent`.
- **Pros:** Rápido de escribir.
- **Contras:** Imposible de mantener, probar o extender. Violación de SRP.

**Opción B — Orquestador + Store + Repositorio (elegida):**
- `BodegasRepository`: única capa HTTP.
- `BodegasStore`: estado reactivo con signals de Angular.
- `BodegasOrchestratorService`: coordina repositorio y store, expone casos de uso.
- Componentes hijo dumb: `@Input` / `@Output` solamente.
- **Pros:** SRP claro, consistente con los demás módulos logísticos, fácil de extender.
- **Contras:** Más archivos, requiere disciplina para no saltarse capas.

---

### Decisión

**Se eligió Opción B.** Consistencia con el patrón establecido en el equipo y facilita la integración futura con un módulo de resumen por bodega que está planificado.

---

### Consecuencias

**Positivas:**
- Cambios de lógica de negocio se hacen en el orquestador sin tocar componentes.
- Los componentes dumb son fáciles de probar en aislamiento.
- El store con signals evita detección de cambios innecesaria.

**Negativas:**
- Mayor cantidad de archivos por módulo.
- El estado del orquestador es mutable (no signals), lo que requiere atención si se usan valores en templates que no actualizan automáticamente.

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-06-24 | Adalberto González | Creación del ADR |
