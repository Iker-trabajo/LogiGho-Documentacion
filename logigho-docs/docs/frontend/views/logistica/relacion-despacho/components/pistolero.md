## 2. Componente: PistoleroComponent

**Selector:** `app-pistolero`  
**Ubicación:** `src/app/views/logistica/relacion-despacho/components/pistolero/pistolero.component.ts`  
**Acceso:** Se muestra únicamente en la pestaña **Hoy**

---

### ¿Qué hace? (para el usuario)

Este componente sirve para ingresar una guía de forma rápida. La persona puede escribirla manualmente o escanearla con la cámara del celular, y luego la envía para que el sistema la procese.

---

### Decoradores

| Decorador | Nombre | Tipo | Descripción |
|---|---|---|---|
| `@Input` | `disabled` | `boolean` | Bloquea el input mientras el padre procesa una guía |
| `@Output` | `guiaEscaneada` | `EventEmitter<string>` | Emite el valor ingresado/escaneado al confirmar con Enter o cámara |

---

### Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `txfiltro` | `string` | Valor actual del input; se limpia después de emitir |
| `isMobile` | `boolean` | Detectado por `userAgent`; cambia el modo de entrada |
| `isActivar` | `boolean` | `true` cuando el scanner de cámara está activo |
| `formatsEnable` | `BarcodeFormat[]` | Formatos aceptados: CODE_128, EAN_13, UPC_A |

---

### Métodos públicos

| Método | Descripción |
|---|---|
| `focus()` | Devuelve el foco al input. El padre lo llama tras cerrar modales para mantener el flujo sin mouse |
| `playBeep()` | Beep de éxito (440 Hz, 100 ms, Web Audio API). Se llama tras pistolerar correctamente |
| `playWarning()` | Sonido de advertencia (80 Hz sawtooth, 2 s). Se llama cuando la guía tiene observaciones |

---

### Flujo principal

```
ngAfterViewInit()
  └─► focus() automático si no es móvil

Usuario escribe y presiona Enter (desktop)
  └─► onKeyPress() → emitir() → guiaEscaneada.emit(valor) + limpia input

Usuario activa cámara y escanea (móvil)
  └─► onScanSuccess() → isActivar = false → emitir()
```

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-03 | Adalberto González | Creación con soporte desktop y móvil (ZXing) |

---

### Observaciones

- `playBeep()` y `playWarning()` usan la Web Audio API (`AudioContext`) para no depender de archivos de audio externos.
- El autofocus usa `setTimeout(..., 50)` para esperar a que Angular termine el ciclo de renderizado antes de hacer focus.
