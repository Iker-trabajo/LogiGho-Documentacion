
## 5. Componente: VistaProveedoresComponent

**Selector:** `app-vista-proveedores`
**Ubicación:** `src/app/views/logistica/resumen-inventario/components/vista-proveedores/vista-proveedores.component.ts`
**Acceso:** Sección inferior de la vista principal de Resumen Inventario

---

### ¿Qué hace? (para el usuario)

Muestra una grilla de tarjetas, una por cada proveedor, con un resumen visual de su inventario:
- Nombre del proveedor y sus iniciales como avatar
- Cantidad de SKUs activos
- Stock actual, unidades en ingresos y en tránsito
- Badges de alerta que indican cuántos productos están "en bajo", "agotados" o en "negativo"

Al pasar el cursor sobre un badge, aparece un tooltip con el nombre e ID de cada producto afectado.

El usuario puede buscar proveedores y ver todos o solo los primeros 8.

---

### Interfaces propias

```typescript
export interface ProductoResumen {
  nombre: string;  // Nombre del producto
  id: string;      // IdProducto como string
}

export interface ResumenProveedor {
  nombre: string;              // Nombre del proveedor/tienda
  skusActivos: number;         // Productos con Actual > 0
  stockActual: number;         // Suma de Actual
  enTransito: number;          // Suma de EnTransito
  ingresos: number;            // Suma de Ingresos
  enBajo: number;              // Cantidad de productos en bajo stock
  agotado: number;             // Cantidad de productos con Actual === 0
  negativo: number;            // Cantidad de productos con Actual < 0
  productosBajo: ProductoResumen[];     // Lista para el tooltip "en bajo"
  productosAgotado: ProductoResumen[]; // Lista para el tooltip "agotado"
  productosNegativo: ProductoResumen[];// Lista para el tooltip "negativo"
}
```

---

### Decoradores y configuración técnica

```typescript
@Component({
  selector: 'app-vista-proveedores',
  standalone: true,
  imports: [CommonModule, FormsModule],
})
export class VistaProveedoresComponent
```

---

### Propiedades clave

**Inputs/Outputs:**

| Propiedad | Tipo | Descripción |
|---|---|---|
| `@Input() set productos` | `ResumenItem[]` | Al llegar datos, dispara `agrupar()` y recalcula los proveedores |
| `@Input() set tiendasProveedor` | `Set<string>` | Si tiene elementos, filtra para mostrar solo esas tiendas |
| `@Input() set umbral` | `number` | Umbral global. Al cambiar, recalcula todos los badges |
| `@Output() filtrarPorProveedor` | `EventEmitter<string>` | Emite el nombre del proveedor al hacer click en su tarjeta |

**Estado interno:**

| Propiedad | Descripción |
|---|---|
| `proveedores` | Lista completa de `ResumenProveedor[]` calculada por `agrupar()` |
| `busquedaProveedor` | Texto del buscador interno |
| `mostrarTodos` | Si `true`, muestra todos los proveedores; si `false`, solo los primeros 8 |
| `LIMITE_VISIBLE` | Constante de 8 tarjetas visibles por defecto |

**Getters:**

| Getter | Descripción |
|---|---|
| `proveedoresVisibles` | Aplica búsqueda y límite de visibilidad sobre `proveedores` |
| `hayMas` | `true` si hay más de 8 proveedores y no hay búsqueda activa |

**Métodos:**

| Método | Descripción |
|---|---|
| `getIniciales(nombre)` | Extrae las iniciales del nombre para el avatar (primera y última palabra) |
| `getEstadoGeneral(p)` | Devuelve `'critico'`, `'alerta'` o `'saludable'` según los contadores del proveedor |
| `agrupar(items)` | Lógica central: agrupa `ResumenItem[]` por `Tienda`, calcula todos los campos de `ResumenProveedor` y ordena por `stockActual` descendente |

---

### Lógica de agrupación

```
Para cada item en productos:
  ├─► Si tiendasProveedor tiene elementos, excluir tiendas que no estén en el Set
  └─► Agrupar por campo "Tienda"

Por cada grupo:
  ├─► bajo     = Actual > 0  AND Actual <= umbral
  ├─► agotado  = Actual === 0
  ├─► negativo = Actual < 0
  └─► Construir ResumenProveedor con contadores y listas de productos

Ordenar por stockActual descendente
```

---

### Tooltips en badges

Cada badge de alerta (`en bajo`, `agotado`, `negativo`) contiene un `div.badge-tooltip` oculto. Al hacer hover, el CSS eleva su `opacity` a 1 y lo hace interactuable. Un pseudo-elemento `::before` crea un puente invisible de 12px que evita que el tooltip desaparezca al mover el mouse desde el badge hacia el tooltip.

---

### Historial de cambios

| Fecha | Cambio |
|---|---|
| 2026-05-26 | Creación de la vista de proveedores con tarjetas agrupadas |
| 2026-05-26 | Agregados tooltips hover en badges con lista de productos afectados |

---

### Observaciones

- Los 3 `@Input() set` (setter) ejecutan `agrupar()` cada vez que cualquiera de las 3 entradas cambia, garantizando que los datos siempre estén sincronizados.
- El `@Input() set tiendasProveedor` permite filtrar la vista para mostrar solo proveedores reales, excluyendo tiendas propias que también aparecen en `ResumenInventario`.
