
## 2. Componente: KpiCardComponent

**Selector:** `app-kpi-card`
**Ubicación:** `src/app/views/logistica/resumen-inventario/components/kpi-card/kpi-card.component.ts`
**Acceso:** Usado internamente por `ResumenInventarioComponent`

---

### ¿Qué hace? (para el usuario)

Es la tarjeta visual que aparece 4 veces en la parte superior de la pantalla. Cada tarjeta muestra:
- Un ícono representativo
- Un número grande (el valor principal del KPI)
- Un título descriptivo
- Un badge opcional con información adicional

Algunas tarjetas son clicables (la de Bajo Stock, por ejemplo) y llevan al usuario a un modal con el detalle.

---

### Decoradores y configuración técnica

```typescript
@Component({
  selector: 'app-kpi-card',
  standalone: true,
  imports: [CommonModule],
})
export class KpiCardComponent
```

Componente presentacional puro: solo recibe datos por `@Input()` y emite eventos por `@Output()`. No tiene lógica de negocio ni llama al backend.

---

### Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `@Input() titulo` | `string` | Etiqueta superior de la tarjeta (ej: "Stock Total") |
| `@Input() valor` | `number \| string` | Valor principal mostrado en grande. Acepta string para casos como "N/A" |
| `@Input() unidad` | `string` | Texto pequeño junto al valor (ej: "unidades") |
| `@Input() badge` | `string` | Texto adicional en el badge inferior |
| `@Input() badgeVariant` | `'verde' \| 'rojo' \| 'naranja' \| 'gris'` | Color del badge |
| `@Input() icono` | `'skus' \| 'stock' \| 'alerta' \| 'transito'` | Determina qué SVG se muestra |
| `@Input() alerta` | `boolean` | Si es `true`, la card cambia a color de advertencia |
| `@Input() clickable` | `boolean` | Si es `true`, muestra cursor pointer y habilita el click |
| `@Output() cardClick` | `EventEmitter<void>` | Evento emitido al hacer click. Solo activo cuando `clickable` es `true` |

---

### Flujo principal

```
Padre pasa valores por @Input()
  └─► Angular renderiza el HTML con título, ícono, valor y badge

Si clickable = true y usuario hace click
  └─► cardClick.emit() → padre abre el modal correspondiente
```

---

### Historial de cambios

| Fecha | Cambio |
|---|---|
| 2026-05-26 | Creación del componente base |

---

### Observaciones

- El componente no sabe qué datos muestra. Solo sabe cómo mostrarlos. El padre decide qué valor y qué color pasan por `@Input()`.
- El modo `alerta` se usa actualmente en la card "Bajo Stock" cuando `productosBajoStock.length > 0`.
