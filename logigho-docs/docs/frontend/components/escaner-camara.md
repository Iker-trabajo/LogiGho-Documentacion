---

## Autor: Adalberto González
Fecha creacion: 2026-07-27  
Estado: desarrollo  
Tipo: componente

# Componente: EscanerCamaraComponent

**Selector:** `app-escaner-camara`   
**Ubicación:** `src/app/components/escaner-camara/escaner-camara.component.ts`  
**Acceso:** Público (reutilizable en cualquier módulo autenticado)

---

## ¿Qué hace?

Muestra un overlay a pantalla completa que activa la cámara del dispositivo y escanea
códigos de barras/QR con `@zxing/ngx-scanner`. Pensado para mobile: un botón abre el
overlay, la cámara se ve dentro de un marco guía, y al detectar un código lo emite y
se cierra solo.

No depende de ningún otro módulo del proyecto: cualquier vista puede usarlo agregándolo
a sus `imports` y escuchando `(scanSuccess)`.

---

## Decoradores

| Decorador | Nombre | Tipo | Descripción |
| --------- | ------ | ---- | ----------- |
| `@Input` | `isOpen` | `boolean` | Abre o cierra el overlay. |
| `@Input` | `formats` | `BarcodeFormat[]` | Formatos de código que reconoce el escáner. Por defecto: `QR_CODE`, `CODE_128`, `EAN_13`, `UPC_A`. |
| `@Input` | `showTorch` | `boolean` | Muestra el botón de linterna cuando el dispositivo la soporta. `true` por defecto. |
| `@Input` | `title` | `string` | Título del header del overlay. `'Escanear código'` por defecto. |
| `@Output` | `scanSuccess` | `EventEmitter<string>` | Emite el valor crudo leído por la cámara al escanear con éxito. |
| `@Output` | `close` | `EventEmitter<void>` | Se dispara al cerrar el overlay (botón ✕, backdrop o `Escape`). |

---

## Propiedades clave

| Propiedad | Tipo | Descripción |
| --------- | ---- | ----------- |
| `torchOn` | `boolean` | `true` mientras la linterna del dispositivo está encendida. |
| `torchDisponible` | `boolean` | `true` si el dispositivo activo reporta soporte de linterna (viene del evento `torchCompatible` de `zxing-scanner`). |
| `permisoDenegado` | `boolean` | `true` si el usuario denegó el permiso de cámara. Mientras esté en `true`, se muestra un mensaje en vez del visor. |

---

## Cómo usarlo en un módulo nuevo

```html
<app-escaner-camara
  [isOpen]="scannerAbierto"
  (scanSuccess)="onGuiaEnter($event)"
  (close)="scannerAbierto = false">
</app-escaner-camara>
```

1. Agrega `EscanerCamaraComponent` al array `imports` del componente que lo va a usar.
2. Controla `isOpen` con una propiedad booleana del componente padre (ej. un botón que solo se muestra en mobile).
3. En `(scanSuccess)`, conecta directo el valor emitido al mismo método que ya procesa una guía escrita a mano o leída con lector USB — no hace falta lógica nueva.

---

## Módulos que lo usan

| Módulo | Uso |
| ------ | --- |
| Relación de Despacho | Botón visible solo en `@media (max-width: 768px)` que abre el overlay; `(scanSuccess)` se conecta directo a `onGuiaEnter()`, el mismo flujo del pistolero de texto/USB. |

---

## Flujo principal

```
El padre pone isOpen = true
  → se renderiza <zxing-scanner>, que pide permiso de cámara al navegador
  → (permissionResponse) informa si el usuario lo concedió o no

Si se concede el permiso
  → se ve el visor de la cámara con el marco guía superpuesto
  → al detectar un código válido: (scanSuccess) → onScan()
        → emite scanSuccess al padre con el valor leído
        → cierra el overlay (onClose)

Si se deniega el permiso
  → permisoDenegado = true
  → se muestra un mensaje pidiendo activar el permiso, en vez del visor

Usuario cierra manualmente (✕, backdrop o Escape)
  → onClose(): resetea torchOn/torchDisponible/permisoDenegado y emite close
```

---

## Historial de cambios

| Fecha | Autor | Cambio |
| ----- | ----- | ------ |
| 2026-07-27 | Adalberto González | Creación del componente reutilizable, extraído del flujo de escaneo por cámara del módulo antiguo `admision-transportadora` (que renderizaba el `<zxing-scanner>` inline en la página, sin overlay ni backdrop). |

---

## Observaciones

- Requiere un contexto seguro (HTTPS o `localhost`) para que `getUserMedia()` funcione: si se prueba desde un celular contra una IP local por HTTP plano, el navegador bloquea el acceso a la cámara sin siquiera mostrar el diálogo de permiso.
- `[torch]`/`(torchCompatible)` de `zxing-scanner` no estaban en uso en ningún otro punto del proyecto antes de este componente; es la primera vez que se controla la linterna del dispositivo.
- El backdrop y el marco guía son puramente decorativos (CSS), sin lógica de negocio.
