---

## Autor: Adalberto González
Fecha creacion: 2026-06-27
Estado: produccion
Tipo: componente

# Componente: TourGuiadoComponent

**Selector:** `app-tour-guiado`
**Ubicación:** `src/app/components/tour-guiado/tour-guiado.component.ts`
**Acceso:** Público (reutilizable en cualquier módulo autenticado)

---

## ¿Qué hace?

Muestra un tour interactivo paso a paso sobre cualquier vista de la aplicación.
Oscurece toda la pantalla excepto el elemento que se quiere explicar, y muestra
una tarjeta flotante con el título y la descripción de ese paso.

No depende de ninguna librería externa. Cada módulo que lo quiera usar solo necesita
decirle qué elementos destacar y qué explicar en cada uno.

---

## Decoradores

| Decorador | Nombre   | Tipo                  | Descripción                                                             |
| --------- | -------- | --------------------- | ------------------------------------------------------------------------ |
| `@Input`  | `isOpen` | `boolean`             | Abre o cierra el tour. Al abrirse siempre empieza desde el primer paso. |
| `@Input`  | `steps`  | `TourStep[]`          | Array de pasos definido directamente en el componente que usa el tour.  |
| `@Output` | `closed` | `EventEmitter<void>`  | Se dispara cuando el usuario cierra el tour, para que el padre lo sepa.  |

### `TourStep`

| Campo | Tipo | Descripción |
| ----- | ---- | ----------- |
| `elementId` | `string` | `id` del elemento HTML a destacar en este paso |
| `title` | `string` | Título corto de la tarjeta |
| `description` | `string` | Explicación — acepta HTML simple (`<b>`, `<br>`) |
| `side?` | `'top' \| 'bottom' \| 'left' \| 'right'` | Dónde aparece la tarjeta respecto al elemento. Default `'bottom'` |
| `onBeforeStep?` | `() => void` | Acción antes de mostrar el paso (ej. abrir un panel que contiene el elemento) |
| `onLeaveStep?` | `() => void` | Acción al salir del paso (ej. cerrar lo que abrió `onBeforeStep`) |

---

## Propiedades clave

| Propiedad      | Tipo                 | Descripción |
| -------------- | -------------------- | ----------- |
| `stepsActivos` | `TourStep[]`         | Los pasos que realmente se muestran. Conserva elementos visibles y pasos con `onBeforeStep`, para revelar un drawer o modal antes de ubicarlo. |
| `stepIndex`    | `number`             | Paso actual, índice sobre `stepsActivos` |
| `pos`          | `PopoverPos \| null` | Posición calculada del spotlight y la tarjeta. `null` mientras se recalcula |

---

## Atajos de teclado

| Tecla    | Qué hace                |
| -------- | ------------------------ |
| `→`      | Ir al siguiente paso      |
| `←`      | Volver al paso anterior   |
| `Escape` | Cerrar el tour            |

---

## Cómo usarlo en un módulo nuevo

1. Pon `id="algo"` en los elementos del HTML que quieras destacar. Si vive dentro de un panel o modal cerrado, usa `onBeforeStep` para abrirlo antes de ubicar el paso.
2. Define un array `readonly tourSteps: TourStep[]` directamente en el `.ts` del componente que va a mostrar el tour (no hace falta un servicio aparte).
3. Agrega `TourGuiadoComponent` al array `imports` del componente raíz del módulo.
4. En el HTML del módulo:

```html
<app-tour-guiado
  [isOpen]="tourAbierto"
  [steps]="tourSteps"
  (closed)="tourAbierto = false">
</app-tour-guiado>
```

---

## Módulos que lo usan

17 archivos en total, entre componentes raíz de vista y modales internos:

| Módulo | Ubicación |
| ------ | --------- |
| Cambio Entrega Inter | `views/logistica/cambio-entrega-inter/` |
| Resumen Inventario | `views/logistica/resumen-inventario/` (+ 3 modales internos) |
| Gestión Bodegas | `views/logistica/gestion-bodegas/` (+ 3 modales internos) |
| Relación Despacho | `views/logistica/relacion-despacho/` (+ carga masiva) |
| Guías Devoluciones | `views/logistica/guias-devoluciones/` |
| Dashboard Sin Despacho | `views/logistica/dashboard-sin-despacho/` |
| Pancake Estadística | `views/tienda/pancake-estadistica/` |
| Gestión Páginas | `views/tienda/gestion-paginas/` |
| Gestión Cuentas | `views/tienda/gestion-cuentas/` |

---

## Flujo principal

```
El padre pone isOpen = true
  -> stepsActivos = conserva elementos existentes y pasos con onBeforeStep
  -> si no hay ninguno, cierra inmediatamente (closed.emit())
  -> stepIndex = 0
  -> irAElemento(): scrollIntoView + sondeo del rect del elemento hasta
     que deja de moverse (requestAnimationFrame, no un delay fijo)
  -> calcPos(): calcula spotlight y tarjeta, cdr.markForCheck()

Al avanzar/retroceder (siguiente()/anterior())
  -> cancela cualquier sondeo de posición anterior todavía en curso
  -> repite scroll + sondeo + cálculo para el nuevo elemento

Al cerrar (X, Finalizar o Escape)
  -> cancela el sondeo pendiente, limpia pos
  -> closed.emit()
```

Si un paso tiene `onBeforeStep`, se ejecuta y se espera 350 ms extra antes de hacer scroll (tiempo para que un modal/panel que abre termine su animación de entrada).

---

## Historial de cambios

| Fecha      | Autor              | Cambio |
| ---------- | ------------------ | ------ |
| 2026-06-27 | Adalberto González | Creación del componente reutilizable, con filtrado automático de pasos por existencia en el DOM. |
| 2026-08-20 | Iker Acevedo | **Fix de posicionamiento:** el sondeo de "¿terminó el scroll?" vigilaba `window.scrollY`/el contenedor con scroll, en vez del elemento destino — si la página terminaba de moverse un instante antes de que el layout del elemento se asentara del todo, la posición calculada quedaba "mirando" el paso anterior. Ahora se sondea el `getBoundingClientRect()` del elemento destino directamente. **Fix de condición de carrera:** ningún sondeo de posición se cancelaba al navegar a otro paso o cerrar el tour — un sondeo viejo que terminaba tarde pisaba la posición/el estado del paso vigente (causaba "a veces avanza y a veces no", "si me devuelvo se daña", "si cierro a la mitad se desconfigura"). Se agregó cancelación real (`cancelAnimationFrame`) más un guard por índice de paso. **Fix de compatibilidad con `OnPush`:** el cálculo de posición corre dentro de un `requestAnimationFrame`, un callback que Angular no propaga automáticamente a través de un componente ancestro con `ChangeDetectionStrategy.OnPush` — el resultado se calculaba bien pero no se pintaba hasta el siguiente evento real, mostrando siempre la posición del paso anterior. Se inyectó `ChangeDetectorRef` y se agregó `markForCheck()` después de cada cálculo asíncrono. **Responsive:** en pantallas ≤480px la tarjeta se ancla fija abajo tipo *bottom-sheet* con scroll interno, en vez de la posición calculada (que asume una altura fija de 160px para no salirse de la ventana — en mobile, con textos largos, esa altura estimada quedaba corta y el pie con los botones "Siguiente"/"Cerrar" terminaba fuera de la pantalla, sin scroll posible). |
| 2026-09-21 | Iker Acevedo Vargas | El tour se recalcula también al hacer scroll, admite pasos que abren un drawer/modal y reserva espacio para explicaciones más largas. Se aplicó al visor de reportes. |

---

## Observaciones

- **El filtrado de pasos es una sola pasada, al abrir.** Si un `elementId` vive dentro de un `*ngIf` que depende de datos cargando asíncronamente (ej. una tabla gateada por `loading()`), y el usuario abre el tour antes de que termine de cargar, ese paso se descarta silenciosamente y el conteo total de pasos baja — no se vuelve a evaluar más tarde. Mitigación usada en `cambio-entrega-inter`: deshabilitar el botón que abre el tour mientras `loading()` es `true`.
- `onBeforeStep` está pensado para revelar un drawer o modal. Debe abrir un elemento con `id` estable y una animación breve; el tour espera a que el layout se asiente antes de destacarlo.
- Se construyó desde cero en lugar de usar driver.js porque el sidebar de CoreUI creaba un conflicto de capas que impedía que el overlay de driver.js se viera correctamente.
- La animación de deslizamiento entre pasos es puro CSS, sin JavaScript de animaciones.
