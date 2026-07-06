---

## Autor: Adalberto González
Fecha creacion: 2026-06-03
Estado: produccion

# Servicio: FilterService

**Ubicación:** `src/app/views/logistica/guias-devoluciones/services/filter.service.ts`
**Scope:** `providedIn: 'root'` Puede ser utilizado en el resto de el proyecto de el front

---

## ¿Qué hace?

Este servicio administra los filtros disponibles en el módulo de Guías de Devoluciones y se encarga de aplicar los criterios de búsqueda sobre la información cargada.

---

## Métodos

### `inicializarFiltroMes(): void`

Puebla el filtro de mes con los últimos 8 meses y preselecciona el mes actual. Se llama en `ngOnInit` del componente.

---

### `setTiendas(tiendas: TiendaInfo[]): void`

Registra las tiendas activas del backend y puebla las opciones de los filtros `ecosistema` y `tienda`.

| Parámetro | Tipo | Descripción |
|---|---|---|
| `tiendas` | `TiendaInfo[]` | Lista de tiendas activas obtenida de la colección `Tienda` |

---

### `actualizarOpcionesTransp(datos: DevolucionRow[]): void`

Actualiza las opciones del filtro `transp` con los valores únicos de transportadora presentes en los datos actuales. Se llama cada vez que `datos` cambia.

| Parámetro | Tipo | Descripción |
|---|---|---|
| `datos` | `DevolucionRow[]` | Registros completos en memoria |

---

### `actualizarFiltroFechas(datos: DevolucionRow[]): void`

Puebla el filtro `fecha` con los días del mes activo que tienen datos. Si hay filtro `tipoDia` activo, solo incluye días que coincidan. Llama internamente a `actualizarFiltroTipoDia`.

| Parámetro | Tipo | Descripción |
|---|---|---|
| `datos` | `DevolucionRow[]` | Registros completos en memoria |

---

### `aplicar(datos: DevolucionRow[]): DevolucionRow[]`

Aplica todos los filtros activos sobre el array de datos y retorna la vista filtrada. El filtro de **mes no se aplica aquí** — se aplica en `ChartComputerService` por prefijo de fecha.

| Parámetro | Tipo | Descripción |
|---|---|---|
| `datos` | `DevolucionRow[]` | Registros completos en memoria |

**Retorna:** Subconjunto filtrado listo para pasar a `ChartComputerService`.

**Orden de filtros aplicados:**
1. Ecosistema → deriva tiendas permitidas
2. Tipo día → filtra por día de la semana
3. Fecha exacta → filtra por `dd/mm/yyyy`
4. Tienda
5. Transportadora
6. Estado (normaliza acentos antes de comparar)

---

### `getPrefixesMesActivo(): Set<string>`

Retorna los prefijos `YYYY-MM` de los meses seleccionados en el filtro de mes. Si no hay selección, retorna los últimos 2 meses.

**Retorna:** `Set<string>` con prefijos como `'2026-06'`, `'2026-05'`.

---

### `getMesSeleccionado(): { year: number; month: number }`

Retorna el año y mes del primer mes seleccionado. Si no hay selección, retorna el mes actual del sistema.

---

### `toggleFilter(key: FilterKey): void`

Abre el filtro indicado y cierra todos los demás.

---

### `toggleOption(key: FilterKey, opt: string): void`

Agrega o quita una opción del filtro. El filtro `mes` permite máximo 2 selecciones simultáneas. El filtro `ecosistema` sincroniza automáticamente las opciones de `tienda` al cambiar.

---

### `removeChip(key: FilterKey, opt: string): void`

Quita una opción específica del filtro (usado al eliminar un chip en la UI).

---

### `clearFilter(key: FilterKey): void`

Limpia selección, búsqueda y cierra un filtro específico.

---

### `clearAll(): void`

Limpia todos los filtros de una vez.

---

### `getFilteredOptions(f: FilterState): string[]`

Retorna las opciones del filtro aplicando el texto de búsqueda actual.

---

### `isSelected(f: FilterState, opt: string): boolean`

Retorna `true` si la opción está seleccionada en el filtro.

---

### `getSelected(key: FilterKey): string[]`

Retorna las opciones seleccionadas de un filtro por su key.

---

### `get hayFiltrosActivos: boolean`

`true` si al menos un filtro tiene opciones seleccionadas. Usado para mostrar el botón "Limpiar todos".

---

## Endpoints que consume

Ninguno. Este servicio no hace HTTP.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-06-03 | Adalberto González | Se creo el service |

---

## Observaciones