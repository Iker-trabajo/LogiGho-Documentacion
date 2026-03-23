---
autor: Iker Acevedo
fecha_creacion: 2026-03-23
estado: aceptada
---

# Implementacion de patrón Strategy para transportadoras en módulo de facturas

**Autor:** Iker Acevedo

**Fecha:** 2026-03-19

**Estado:** Produccion

---

## Contexto

El módulo `facturas-trans` gestiona las los pagos realizados a las diferentes transportadoras. Al momento de esta decisión existían 3 transportadoras implementadas (Interrapidisimo, Envia, X-Cargo) y se requería agregar Servientrega.

El patrón de implementación existente duplicaba el mismo flujo (fetch → descomprimir → sanitizar → mostrar → exportar → importar) para cada transportadora, con variables y métodos individuales por cada una. Agregar Servientrega con el mismo patrón implicaba ~200 líneas duplicadas adicionales y un componente de importación de ~550 líneas.

**Restricciones:**
- No se puede refactorizar el codigo hasta que se inicie el proyecto dentro del modulo.
- Servientrega debía entregarse en el ciclo actual, sin afectar al actual
- Las transportadoras futuras deben poder agregarse con el mínimo impacto posible

---

## Opciones consideradas

### Opción A — Continuar con el patrón legacy (duplicación)
Agregar Servientrega igual que las 3 transportadoras existentes: variables propias, métodos propios, componente de importación propio.

**Pros:**
- Sin curva de aprendizaje para el equipo
- Consistente con lo que ya existe

**Contras:**
- ~200 líneas duplicadas por transportadora nueva
- Un componente de importación de ~550 líneas por cada una
- Un bug en el batch insert hay que corregirlo en N componentes
- Con 6-8 transportadoras el componente supera las 2000 línea
- Se vuelve inmantenible para la empresa un componente tan pesado y muy dificil de actualizar o mejorar

---

### Opción B — Patrón Strategy solo para transportadoras nuevas con proyecto de refactorizar actuales
Implementar Servientrega con Strategy sin modificar las 3 existentes. Las transportadoras legacy coexisten con las nuevas hasta que el equipo evalúe la migración.

**Pros:**
- Cero riesgo sobre código existente en producción
- Servientrega sirve como referencia y validación del patrón
- Transportadoras futuras siguen el flujo de Servientrega
- La migración de las legacy queda como decisión separada del equipo

**Contras:**
- Coexisten dos patrones en el mismo componente temporalmente
- Un dev nuevo debe entender ambos estilos

---

## Decisión

**Se eligió:** Opción N

**Razón principal:** Permitía entregar Servientrega con buenas prácticas sin poner en riesgo las 3 transportadoras que ya funcionan en producción. Servientrega actúa como prueba del patrón. Si el equipo valida la arquitectura, la migración de las legacy es un proceso seguro y evaluado.

---

## Consecuencias

### Positivas
- Agregar una transportadora nueva requiere: 1 clase nueva + 1 línea en el factory + copiar bloque HTML
- `app-importacion-facturas-generico` sirve para todas las transportadoras nuevas sin duplicación
- Un solo lugar para corregir bugs en la lógica de importación
- Contrato claro: cualquier dev sabe exactamente qué implementar para agregar una transportadora

### Negativas / Compromisos
- El componente `facturas-trans` tiene temporalmente dos patrones conviviendo (legacy + Strategy)
- Las variables individuales de las legacy (`rows`, `rowsEnvia`, `rowsXcargo`) coexisten con el `tablaEstadoMap`
- La migración de las legacy queda como deuda técnica hasta que el equipo la priorice

### Neutrales
- El `TransportadoraFactory` centraliza la creación de estrategias pero no es obligatorio usarlo directamente en el template

---

## Implementación

| Módulo / Repo | Impacto |
|---|---|
| `SitioLogiGho` — `facturas-trans` | Nuevos archivos en `strategies/transportadoras/`. Métodos genéricos agregados al componente. Sección Servientrega en el HTML. |
| `SitioLogiGho` — `components` | Nuevo componente `importacion-facturas-generico` que reemplaza el patrón de un componente por transportadora |

**Archivos creados:**
- `strategies/transportadoras/transportadora.strategy.ts`
- `strategies/transportadoras/transportadora.factory.ts`
- `strategies/transportadoras/servientrega.strategy.ts`
- `components/importacion-facturas-generico/`

---

## Estado futuro

Esta decisión se revisará si:
- El equipo aprueba migrar las transportadoras legacy (Interrapidisimo, Envia, X-Cargo) al patrón Strategy
- Se agrega una transportadora que requiera un flujo radicalmente diferente al cubierto por la interfaz actual
- La interfaz `TransportadoraStrategy` crece con campos que no aplican a todas las transportadoras — en ese caso evaluar separar en dos interfaces: `TransportadoraDisplayStrategy` e `TransportadoraImportStrategy`

---

