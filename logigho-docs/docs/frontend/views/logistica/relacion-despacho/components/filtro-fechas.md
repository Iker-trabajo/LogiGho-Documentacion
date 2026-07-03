## 4. Componente: FiltroFechasComponent

**Selector:** `app-filtro-fechas`  
**Ubicación:** `src/app/views/logistica/relacion-despacho/components/filtro-fechas/filtro-fechas.component.ts`  
**Acceso:** Se muestra únicamente en la pestaña **Histórico**

---

### ¿Qué hace? (para el usuario)

Este componente ayuda a elegir un rango de fechas para ver el historial de despachos. Así la persona puede buscar lo que pasó en días anteriores y ver la información de forma ordenada.

---

### Decoradores

| Decorador | Nombre | Tipo | Descripción |
|---|---|---|---|
| `@Input` | `fechaInicio` | `string` | Fecha de inicio pre-seleccionada (`YYYY-MM-DD`) |
| `@Input` | `fechaFin` | `string` | Fecha de fin pre-seleccionada (`YYYY-MM-DD`) |
| `@Output` | `buscar` | `EventEmitter<RangoFechas>` | Emite `{inicio, fin}` al hacer clic en Buscar |
| `@Output` | `limpiar` | `EventEmitter<void>` | Emite al limpiar; el padre vacía `rowsHistorico` |

---

### Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `activeCalendar` | `'inicio' \| 'fin' \| null` | Campo activo del calendario abierto |
| `calendarNav` | `{month, year}` | Mes y año visible en el dropdown del calendario |
| `calendarDays` | `Date[]` | Días generados para la grilla (incluye días de meses adyacentes) |
| `dropdownStyle` | `{top, left}` | Posición `fixed` calculada al abrir el calendario |

---

### Flujo principal

```
Usuario hace clic en un campo de fecha
  └─► openCalendar() → calcula posición fixed → genera días del mes

Usuario navega meses (← →)
  └─► prevMonth() / nextMonth() → generateCalendarDays()

Usuario selecciona un día
  └─► selectDay() → asigna fechaInicio o fechaFin → cierra calendario

Usuario hace clic en Buscar
  └─► onBuscar() → buscar.emit({inicio, fin}) → padre llama cargarHistorico()

Usuario hace clic en Limpiar
  └─► onLimpiar() → limpia fechas → limpiar.emit() → padre vacía rowsHistorico
```

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-03 | Adalberto González | Creación con calendario custom y posicionamiento fixed |

---

### Observaciones

- El calendario usa `position: fixed` calculado desde `getBoundingClientRect()` para no quedar oculto por `overflow: hidden` de contenedores padres.
- `generateCalendarDays()` rellena la primera fila con días del mes anterior si el mes no comienza en domingo, garantizando siempre filas de 7 columnas.
- `@HostListener('document:click')` cierra el calendario al hacer clic fuera; el wrapper usa `stopPropagation` para evitar que el clic de apertura lo cierre inmediatamente.
