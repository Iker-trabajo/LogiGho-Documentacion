---

## Autor: Adalberto González
Fecha creacion: 2026-06-27  
Estado: desarrollo  
Tipo: componente

# Componente: GuidedTourComponent

**Selector:** `app-guided-tour`   
**Ubicación:** `src/app/components/guided-tour/guided-tour.component.ts`  
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

| Decorador | Nombre   | Tipo                 | Descripción                                                                 |
| --------- | -------- | -------------------- | --------------------------------------------------------------------------- |
| `@Input`  | `isOpen` | `boolean`            | Abre o cierra el tour. Al abrirse siempre empieza desde el primer paso.     |
| `@Input`  | `steps`  | `TourStep[]`         | La lista de pasos que viene del servicio del módulo que usa el tour.        |
| `@Output` | `closed` | `EventEmitter<void>` | Se dispara cuando el usuario cierra el tour, para que el padre lo sepa.     |

---

## Propiedades clave

| Propiedad      | Tipo          | Descripción                                                                                                    |
| -------------- | ------------- | -------------------------------------------------------------------------------------------------------------- |
| `stepsActivos` | `TourStep[]`  | Los pasos que realmente se van a mostrar. Al abrir el tour se filtran automáticamente los que no tienen su elemento en pantalla (por ejemplo, botones que solo ven ciertos roles). |
| `stepIndex`    | `number`      | Qué paso se está mostrando ahora mismo.                                                                        |
| `pos`          | `PopoverPos \| null` | Dónde está posicionada la tarjeta y el recorte en pantalla. Es `null` mientras la página está haciendo scroll. |

---

## Atajos de teclado

| Tecla    | Qué hace                    |
| -------- | --------------------------- |
| `→`      | Ir al siguiente paso        |
| `←`      | Volver al paso anterior     |
| `Escape` | Cerrar el tour              |

---

## Cómo usarlo en un módulo nuevo

1. Pon `id="algo"` en los elementos del HTML que quieras destacar.
2. Crea un servicio con la lista de pasos (mira `ResumenInventarioTourService` como ejemplo).
3. Agrega `GuidedTourComponent` al array `imports` del componente raíz del módulo.
4. En el HTML del módulo, pega esto:

```html
<app-guided-tour
  [isOpen]="tourAbierto"
  [steps]="tourService.steps"
  (closed)="tourAbierto = false">
</app-guided-tour>
```

---

## Módulos que lo usan

| Módulo                | Servicio de pasos               |
| --------------------- | ------------------------------- |
| Resumen de Inventario | `ResumenInventarioTourService`  |
| Gestión de Bodegas    | `GestionBodegasTourService`     |

---

## Flujo principal

```
El padre pone isOpen = true
  → filtra los pasos cuyos elementos existen en pantalla
  → si no hay ninguno, cierra inmediatamente
  → hace scroll al primer elemento
  → espera 420 ms (el scroll suave tarda ese tiempo)
  → calcula la posición y muestra la tarjeta

Al avanzar o retroceder
  → repite el scroll + espera + cálculo para el nuevo elemento

Al cerrar (✕, Finalizar o Escape)
  → avisa al padre para que ponga isOpen = false
```

---

## Historial de cambios

| Fecha      | Autor              | Cambio                                                                          |
| ---------- | ------------------ | ------------------------------------------------------------------------------- |
| 2026-06-27 | Adalberto González | Creación de el componente reutilizable. Se agrega el filtrado automático de pasos por rol. |

---

## Observaciones

- El delay de 420 ms antes de calcular la posición es intencional: si se calcula antes de que el scroll termine, la tarjeta aparece en el lugar equivocado.
- Se construyó desde cero en lugar de usar driver.js porque el sidebar de CoreUI creaba un conflicto de capas que impedía que el overlay de driver.js se viera correctamente.
- La animación de deslizamiento entre pasos es puro CSS, sin JavaScript de animaciones.
