## Servicio: devoluciones.rules (funciones puras)

**Autor:** Adalberto González
**Fecha:** 2026-07-28
**Estado:** producción
**Ubicación:** `src/app/views/logistica/guias-devoluciones/helpers/devoluciones.rules.ts`
**Scope:** Módulo utilitario — funciones puras, sin clase ni `@Injectable`

---

## ¿Qué hace?

Reemplaza la lógica de cálculo puro que antes vivía en `FilterService`. Funciones sin estado ni efectos secundarios, usadas por `DevolucionesStore`.

---

## Métodos

### `buildFiltrosAgg(filters)`

Empaqueta el estado de filtros/mes en la forma que espera el `agregacion.worker`.

### `getPrefixesMesActivo(filters)`

Prefijos `YYYY-MM` del mes activo. Sin selección, retorna los últimos 2 meses.

### `estadoBadge(estado): {label, clase}`

Etiqueta capitalizada y clase CSS del badge de estado en la tabla de detalle (Completada/Ratificada/Regional/Devolución).

### `tiendaCorta(nombre): string`

Nombre de tienda acortado a 2 palabras + "…" si es más largo, con el nombre completo en el tooltip.

> El resto (`filtrosIniciales`, `opcionesMes`, `fechasConDatos`, `opcionesTipoDia`, `opcionesEcosistemaYTienda`) reemplazan a los métodos equivalentes de `FilterService`, sin estado propio.

---

## Endpoints que consume

Ninguno. Módulo puro.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-28 | Adalberto González | Creación, extrayendo la lógica pura de `FilterService` y agregando `estadoBadge`/`tiendaCorta` para el parseo visual de la tabla de detalle |

---

## Observaciones

- El componente expone `estadoBadge()`/`tiendaCorta()` como métodos delgados para poder llamarlas desde el template.
