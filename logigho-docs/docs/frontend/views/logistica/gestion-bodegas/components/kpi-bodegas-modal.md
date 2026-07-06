## Componente: KpiBodegasModalComponent

**Selector:** `app-kpi-bodegas-modal`  
**Ubicación:** `src/app/views/logistica/gestion-bodegas/components/kpi-bodegas-modal/kpi-bodegas-modal.component.ts`  
**Acceso:** Se abre al hacer clic en cualquiera de las 3 tarjetas KPI de la vista principal

---

### ¿Qué hace? (para el usuario)

Muestra un modal con el listado de bodegas correspondiente al KPI seleccionado. Por ejemplo, al hacer clic en "Bodegas Bajo Stock" muestra solo las bodegas que están por debajo del umbral de inventario. Para cada bodega muestra nombre, ID, ubicación, badge de estado y stock total.

---

### Decoradores y configuración técnica

```typescript
@Component({
  selector: 'app-kpi-bodegas-modal',
  standalone: true,
  imports: [CommonModule],
})
export class KpiBodegasModalComponent
```

Componente dumb: recibe la lista ya filtrada desde el padre. No filtra ni calcula internamente.

---

### Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `@Input() isOpen` | `boolean` | Controla visibilidad |
| `@Input() titulo` | `string` | Título dinámico según el KPI clickeado |
| `@Input() subtitulo` | `string` | Descripción del contexto (ej: "Stock por debajo del umbral mínimo") |
| `@Input() bodegas` | `Bodega[]` | Lista ya filtrada de bodegas a mostrar |
| `@Input() stockPorBodega` | `Map<string, number>` | `idBodega → stockTotal` para mostrar en cada fila |
| `@Output() cerrar` | `EventEmitter<void>` | Cierra el modal |

---

### Flujo principal

```
Usuario hace clic en una tarjeta KPI
  └─► padre llama abrirKpiDetalle(tipo)
        └─► filtra store.bodegas() según tipo:
              'todas'     → todas las bodegas
              'stock'     → bodegas activas con su stock
              'bajoStock' → bodegas con stock < 5000 (incluye las sin asignaciones)
        kpiDetalleBodegas = resultado filtrado
        kpiDetalleAbierto = true → isOpen = true → modal visible

Usuario cierra
  └─► cerrar.emit() → padre pone kpiDetalleAbierto = false
```

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-06-24 | Adalberto González | Creación del modal de detalle KPI exclusivo para bodegas |
| 2026-06-25 | Adalberto González | Filas clickeables navegan a resumen-inventario; deshabilitadas si la bodega no tiene asignaciones activas |

---

### Observaciones

- Distinto al `kpi-detalle-modal` de `resumen-inventario`, que muestra productos con imágenes y tiendas. Este muestra bodegas con su stock físico.
- Cierra con tecla Escape y clic en el overlay.
- Las bodegas sin ninguna asignación activa aparecen con stock `0` en el listado de bajo stock, ya que ese es su estado real.
