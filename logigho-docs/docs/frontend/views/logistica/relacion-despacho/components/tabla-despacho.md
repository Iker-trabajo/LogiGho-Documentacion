## 3. Componente: TablaDespachoComponent

**Selector:** `app-tabla-despacho`  
**Ubicación:** `src/app/views/logistica/relacion-despacho/components/tabla-despacho/tabla-despacho.component.ts`  
**Acceso:** Siempre visible en la vista principal

---

### ¿Qué hace? (para el usuario)

Este componente muestra la lista de guías en forma de tabla para que sea más fácil verlas. Puede mostrar las guías del día o las del historial, y también permite seleccionar algunas con una casilla para trabajar con ellas después.

---

### Decoradores

| Decorador | Nombre | Tipo | Descripción |
|---|---|---|---|
| `@Input` | `rows` | `RegistroDespacho[]` | Guías para la pestaña Hoy (ya filtradas por el padre) |
| `@Input` | `rowsHistorico` | `RegistroDespacho[]` | Guías para la pestaña Histórico (ya filtradas por el padre) |
| `@Input` | `activeTab` | `'Hoy' \| 'historico'` | Determina qué tabla se renderiza |
| `@Input` | `columns` | `ModernTableColumn[]` | Definición de columnas compartida con el padre |
| `@Output` | `tabChange` | `EventEmitter<'Hoy' \| 'historico'>` | No usado actualmente; reservado para futura navegación desde tabla |
| `@Output` | `seleccionCambio` | `EventEmitter<RegistroDespacho[]>` | Propaga la selección de checkboxes al padre |

---

### Flujo principal

```
Padre pasa rows/rowsHistorico ya filtrados
  └─► TablaDespacho renderiza ModernTable correspondiente
        └─► Usuario selecciona checkboxes
              └─► seleccionCambio.emit() → padre actualiza seleccionadas[]
```

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-03 | Adalberto González | Creación como contenedor delgado con selección de guías |

---

### Observaciones

- Este componente no tiene lógica propia; es un puente entre la vista padre y `ModernTableComponent`.
