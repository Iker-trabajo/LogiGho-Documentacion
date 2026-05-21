---

## Autor: Adalberto González  
Fecha creacion: 2026-05-20  
Estado: aceptada

# ADR-001 — Patrón Template Method para validación de carga masiva de productos

**Autor:** Adalberto González  
**Fecha:** 2026-05-20  
**Estado:** Aceptada

---

## Contexto

El módulo de carga masiva de productos (`app-creaciones-masiva-productos`) necesita validar cada fila de un Excel antes de insertarla en MongoDB. Las validaciones tienen dos capas:

1. **Estructural / referencial** — verificar que los campos obligatorios existan, que los IDs de Tienda, Categoría y Proveedor referencien documentos reales en BD, y que los tipos de dato sean correctos.
2. **De negocio** — reglas que pueden cambiar por decisión comercial (ej. `precioventa > precioproveedor`, tiendas asignadas al usuario).

El componente inicial tenía toda esa lógica dentro del propio `procesarExcel()`, lo que generaba un método de más de 80 líneas difícil de mantener y de extender sin romper lo que ya funcionaba.

Restricciones:
- Nuevas reglas de negocio se agregan frecuentemente (el equipo puede añadir restricciones por tienda, por categoría o por rol en cualquier sprint).
- El equipo frontend no quiere tocar el componente cada vez que se añade una regla.
- Angular standalone no permite usar herencia de servicios inyectables de forma simple; las clases POJO son más directas.

---

## Opciones consideradas

### Opción A — Lógica inline en el componente

Toda la validación en un único método del componente, con `if` acumulativos.

**Pros:** Nada que importar, fácil de leer con pocas reglas.  
**Contras:** El método crece con cada regla; mezcla responsabilidades (parsing, validación estructural, validación de negocio); sin contrato que fuerce consistencia al añadir reglas.

### Opción B — Strategy Pattern (una clase por regla)

Cada regla implementa una interfaz `IReglaNegocio` con un método `evaluar(fila): string[]`. El componente itera un array de estrategias.

**Pros:** Muy extensible; se puede habilitar/deshabilitar una regla cambiando el array.  
**Contras:** Overhead de clases pequeñas con una sola línea de lógica; el contexto (tiendas, categorías) se pasa a cada estrategia, lo que ensucia la interfaz o exige un objeto de contexto grande.

### Opción C — Template Method Pattern (clase base + hook de negocio)

Una clase abstracta `ImportacionMasivaBase` implementa el algoritmo completo de evaluación (validación estructural, resolución de IDs, construcción del objeto `FilaPreview`) y expone un único método abstracto `validarNegocio(raw, ctx)` que los subtipos concretos implementan.

**Pros:** La secuencia de validación está garantizada por la clase base (nunca se puede omitir una fase); el hook de negocio es el único punto a cambiar; el contrato es mínimo (firma fija); fácil de testear unitariamente la clase base y el validator por separado.  
**Contras:** Herencia, que puede ser rígida si en el futuro se necesitan múltiples algoritmos base distintos. Con un único dominio (productos) esto no es un problema real hoy.

---

## Decisión

**Se eligió:** Opción C — Template Method Pattern

**Razón:** El algoritmo de validación tiene pasos fijos (estructural primero, luego de negocio) y un único punto de variación (las reglas de negocio). El Template Method modela exactamente ese contrato: la clase base garantiza el orden y el subtipo concreto solo aporta la variabilidad. Con Strategy habría que coordinar el orden entre estrategias y manejar el contexto en cada una; aquí el contexto ya está encapsulado en `ContextoValidacion` y se pasa una sola vez.

---

## Consecuencias

**Positivas:**
- Agregar una nueva regla de negocio requiere solo editar `validarNegocio()` en `ProductoImportacionValidator`, sin tocar el componente ni la clase base.
- La clase base es estable; los tests de integración no se rompen al añadir reglas.
- El contrato `(raw, ctx) → string[]` es predecible y fácil de documentar.
- Sigue la misma convención de carpeta que `facturas-trans/strategies/`, lo que reduce la curva de aprendizaje para el equipo.

**Negativas:**
- Si en el futuro se necesitan dos algoritmos base distintos (ej. validación de servicios vs. productos), se requiere una segunda clase base o refactor a Strategy.
- La herencia implícita oculta el flujo completo a quien solo lee `ProductoImportacionValidator`; hay que navegar a la clase base para entender el algoritmo.

---

## Impacto en el código

| Módulo / Repo  | Cambio |
| -------------- | ------ |
| `SitioLogiGho` | Nueva carpeta `portafolio/importacion-template-method/` con 4 archivos: `importacion-masiva.base.ts`, `importacion-masiva.types.ts`, `producto-importacion.validator.ts`, `PLANTILLA-ADR.md` |
| `SitioLogiGho` | `CreacionesMasivaProductosComponent` delega toda la evaluación de filas a `ProductoImportacionValidator.evaluarFila()` |
| `LambdasLogiGho` | Sin cambios — la lógica es puramente frontend |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-05-20 | Adalberto González | Creación del ADR. Implementación inicial con Regla 1 (`precioventa > precioproveedor`) y Regla 2 (`IdTienda` en tiendas asignadas al usuario) activas. |

---

## Referencias

- Clase base: `src/app/views/tienda/portafolio/importacion-template-method/importacion-masiva.base.ts`
- Validator concreto: `src/app/views/tienda/portafolio/importacion-template-method/producto-importacion.validator.ts`
- Tipos y constantes: `src/app/views/tienda/portafolio/importacion-template-method/importacion-masiva.types.ts`
- Componente consumidor: `src/app/components/creaciones-masiva-productos/creaciones-masiva-productos.component.ts`
- Patrón análogo en el proyecto: `src/app/views/tienda/facturas-trans/strategies/`
