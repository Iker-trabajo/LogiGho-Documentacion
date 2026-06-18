
## 6. Componente: FiltrosInventarioComponent

**Selector:** `app-filtros-inventario`
**Ubicación:** `src/app/views/logistica/resumen-inventario/components/filtros-inventario/filtros-inventario.component.ts`
**Acceso:** Header de la vista principal de Resumen Inventario

---

### ¿Qué hace? (para el usuario)

Provee dos dropdowns de filtro en el header de la pantalla:

1. **Filtro por Ecosistema:** El usuario puede desplegar un panel con todos los ecosistemas de LogiGho y marcar/desmarcar grupos de tiendas. Al seleccionar un ecosistema completo, se marcan todas sus tiendas automáticamente.

2. **Filtro por Tienda:** Permite seleccionar tiendas individuales de cualquier ecosistema. Tiene un buscador interno.

3. **Filtro de Estado:** Permite mostrar productos Activos, Inactivos o Todos.

Un badge numérico indica cuántos filtros de tienda están activos. El botón "Limpiar" resetea todo.

---

### Decoradores y configuración técnica

```typescript
@Component({
  selector: 'app-filtros-inventario',
  standalone: true,
  imports: [CommonModule, FormsModule],
})
export class FiltrosInventarioComponent
```

Usa `@HostListener('document:click')` para cerrar los dropdowns al hacer click fuera de ellos.

---

### Propiedades clave

**Inputs/Outputs:**

| Propiedad | Tipo | Descripción |
|---|---|---|
| `@Input() set tiendaMap` | `Map<string, string[]>` | Al llegar el mapa, extrae y ordena los ecosistemas alfabéticamente |
| `@Input() estadoActual` | `string` | Estado actualmente seleccionado (para sincronía con el padre) |
| `@Output() filtroChange` | `EventEmitter<Set<string>>` | Emite el nuevo Set de tiendas seleccionadas ante cualquier cambio |
| `@Output() estadoChange` | `EventEmitter<string>` | Emite el nuevo estado seleccionado |

**Estado interno:**

| Propiedad | Descripción |
|---|---|
| `_tiendaMap` | Copia interna del mapa recibido |
| `ecosistemas` | Array de nombres de ecosistemas ordenados |
| `tiendasLocal` | Set con los nombres de tiendas marcadas por el usuario |
| `showEcoDropdown / showTiendaDropdown` | Visibilidad de cada dropdown |
| `searchEco / searchTienda` | Texto del buscador en cada dropdown |

**Getters:**

| Getter | Descripción |
|---|---|
| `ecosistemasFiltrados` | Ecosistemas que coinciden con el texto de búsqueda |
| `todasLasTiendas` | Todas las tiendas del mapa, sin repetir, ordenadas |
| `tiendasFiltradas` | Tiendas que coinciden con el texto de búsqueda |
| `totalFiltros` | Número de tiendas marcadas actualmente |

**Métodos:**

| Método | Descripción |
|---|---|
| `getEcoEstado(eco)` | Devuelve `'all'`, `'some'` o `'none'` para el estado del checkbox del ecosistema |
| `isTiendaSeleccionada(nombre)` | `true` si la tienda está en `tiendasLocal` |
| `toggleEcosistema(eco)` | Marca o desmarca todas las tiendas de un ecosistema |
| `toggleTienda(nombre)` | Marca o desmarca una tienda individual |
| `limpiarFiltros()` | Vacía `tiendasLocal` y emite el cambio |
| `emitir()` (privado) | Crea un nuevo `Set` con el estado actual y lo emite por `filtroChange` |

---

### Flujo principal

```
Padre pasa tiendaMap por @Input()
  └─► set tiendaMap dispara extracción de ecosistemas ordenados

Usuario despliega dropdown de ecosistemas
  └─► showEcoDropdown = true (el click fuera lo cierra via @HostListener)

Usuario marca ecosistema "Bogotá"
  └─► toggleEcosistema("Bogotá") → marca todas sus tiendas en tiendasLocal
      emitir() → filtroChange.emit(new Set(tiendasLocal))
        └─► Padre actualiza tiendasSeleccionadas
            datosVista getter recalcula
```

---

### Historial de cambios

| Fecha | Cambio |
|---|---|
| 2026-05-26 | Creación con dropdowns de ecosistema y tienda con buscador |

---

### Observaciones

- `emitir()` siempre crea un `new Set(tiendasLocal)` para que Angular detecte el cambio por referencia, no por mutación.
- `@HostListener('document:click')` usa `document.getElementById()` para comparar si el click fue dentro o fuera de cada wrapper de dropdown. Esto es necesario porque el contenido proyectado no siempre está en el árbol de hijos del componente.
