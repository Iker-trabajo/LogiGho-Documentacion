---

## Autor: Adalberto González
Fecha creacion: 2026-07-28
Estado: produccion

# Servicio: devoluciones.rules (funciones puras)

**Ubicación:** `src/app/views/logistica/guias-devoluciones/helpers/devoluciones.rules.ts`
**Scope:** Módulo utilitario — funciones puras, sin clase ni `@Injectable`

---

## ¿Qué hace?

Reemplaza la lógica de cálculo puro que antes vivía en `FilterService`. Funciones sin estado ni efectos secundarios, usadas por `DevolucionesStore`.

---

## Métodos

### `buildFiltrosAgg(filters: FilterState[]): DevolucionesFiltrosAgg`

Empaqueta el estado de filtros/mes en la forma que espera el `agregacion.worker`.

### `getPrefixesMesActivo(filters: FilterState[]): Set<string>`

Prefijos `YYYY-MM` del mes activo. Sin selección, retorna los últimos 2 meses.

### `estadoBadge(estado: string): {label, clase}`

Etiqueta capitalizada y clase CSS del badge de estado en la tabla de detalle.

### `tiendaCorta(nombre: string): string`

Nombre de tienda acortado a 2 palabras + "…" si es más largo.

> Ver el archivo fuente para el resto (`filtrosIniciales`, `opcionesMes`, `fechasConDatos`, `opcionesTipoDia`, `opcionesEcosistemaYTienda`, `opcionesTiendaPorEco`) — todas siguen el mismo patrón: reciben el estado como parámetro, sin leer nada global.

---

## Endpoints que consume

Ninguno. Módulo puro.

---

## Historial de cambios

| Fecha      | Autor              | Cambio                                                                                          |
| ----------- | -------------------- | -------------------------------------------------------------------------------------------- |
| 2026-07-28 | Adalberto González | Creación, extrayendo la lógica pura de `FilterService` y agregando `estadoBadge`/`tiendaCorta` |

---

## Observaciones

- El componente expone `estadoBadge()`/`tiendaCorta()` como métodos delgados para poder llamarlos desde el template.
