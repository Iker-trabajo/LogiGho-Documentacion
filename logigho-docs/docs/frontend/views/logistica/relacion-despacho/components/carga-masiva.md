## 5. Componente: CargaMasivaComponent

**Selector:** `app-carga-masiva`  
**Ubicación:** `src/app/views/logistica/relacion-despacho/components/carga-masiva/carga-masiva.component.ts`  
**Acceso:** Se abre desde el botón "Carga masiva" o con el atajo `Ctrl + M`

---

### ¿Qué hace? (para el usuario)

Este componente permite subir un archivo con muchas guías a la vez. El sistema revisa cada una, muestra cuáles están bien y cuáles no, y luego las prepara para registrarlas de forma rápida.

---

### Decoradores

| Decorador | Nombre | Tipo | Descripción |
|---|---|---|---|
| `@Input` | `isOpen` | `boolean` | Controla la visibilidad del modal |
| `@Output` | `close` | `EventEmitter<void>` | Emite al cerrar (botón ✕, backdrop o `Escape`) |
| `@Output` | `registrosInsertados` | `EventEmitter<RegistroDespacho[]>` | Emite los registros válidos para que el padre los persista |

---

### Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `selectedFile` | `File \| null` | Archivo seleccionado por el usuario |
| `isLoading` | `boolean` | `true` mientras se procesa; bloquea el cierre con `Escape` |
| `progreso` | `number` | Porcentaje 0–100 para la barra de progreso |
| `preview` | `PreviewDespacho[]` | Resultados de validación listos para mostrar en tabla |

---

### Interfaz PreviewDespacho

```typescript
export interface PreviewDespacho {
  Guia: string;        // Número de guía como string (antes de convertir a Number)
  NombreTienda: string;
  EstadoGuia: string;  // Estado del pedido en PedidosInter
  Validacion: string;  // 'OK' o motivo de rechazo (puede ser compuesto con ' | ')
  Cliente: string;
  Departamento: string;
  TotalRecaudo: string;
  Productos: string[];
  Fecha: string;
  pedidoId?: string;   // _id en PedidosInter para fire-and-forget
  _idStocks?: number[];// IdStock1..12 extraídos durante validación (evita segunda consulta)
}
```

---

### Flujo principal

```
Usuario abre modal (botón o Ctrl+M)
  └─► isOpen = true → modal visible

Usuario selecciona archivo Excel/CSV
  └─► onFileSelected() → lee con XLSX → extrae columna "Guia"

Usuario hace clic en "Cargar archivo"
  └─► procesarArchivo() [por cada guía en paralelo controlado]
        ├─► GET PedidosInter (buscar pedido)
        ├─► validar: guía existe, tienda asignada, no duplicada
        ├─► GET Productos (por IdStock1..12)
        └─► acumula en preview[]

Usuario revisa preview y confirma
  └─► registrosInsertados.emit(registrosValidos)
        └─► padre: POST InventarioAdmision + fire-and-forget PUT PedidosInter

Usuario cierra modal
  └─► close.emit() → padre: showCargaMasiva = false
```

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-03 | Adalberto González | Creación con validación por guía, preview y barra de progreso |

---

### Observaciones

- La **inserción en BD ocurre en el padre** (`onMasivaCompleta`), no aquí. Este componente solo valida y emite. Así se mantiene el principio de que solo la vista raíz hace escrituras.
- `_idStocks` se guarda en `PreviewDespacho` durante la validación para evitar una segunda consulta a `PedidosInter` al construir el `RegistroDespacho` final.
- El cierre con `Escape` está bloqueado cuando `isLoading = true` para evitar que el usuario interrumpa una carga en progreso.
