---

## Autor: Yadilton Torres
Fecha creacion: 2026-10-14
Estado: desarrollo

# Directiva: CampoSoloNumerosDirective

**Selector:** `[appCampoSoloNumeros]`  
**Ubicación:** `src/app/views/tienda/pauta/directives/campo-solo-numeros.directive.ts`  
**Tipo:** `standalone`

---

## ¿Qué hace?

Hace que un input solo acepte dígitos: bloquea puntos, comas, espacios, letras y signos **al escribir y al pegar**, y muestra un aviso con [`AvisoToastService`](../services/aviso-toast-service.md) explicando por qué. Los montos de pauta no usan decimales ni separadores de miles, así que se escribe `1500000`, no `1.500.000`.

---

## Uso

```html
<!-- Siempre activa -->
<input appCampoSoloNumeros [(ngModel)]="item.Meta" />

<!-- Activa solo si se cumple la condición (con false queda apagada) -->
<input [appCampoSoloNumeros]="COLUMNAS_NUMERICAS.includes(col)" />
```

| Dónde se usa | Campos |
|---|---|
| [Vista Pauta](../pauta.md) | Meta, TikTok, Google y Otros al editar una fila |
| [Importacion Pauta](../../../../components/importacion-pauta.md) | Idtienda y montos en la previsualización |

---

## Entradas y eventos

| Decorador | Nombre | Tipo | Descripción |
|---|---|---|---|
| `@Input` | `appCampoSoloNumeros` | `boolean \| ''` | Activa la validación. Solo el atributo (vacío) cuenta como activo |
| `@HostListener` | `beforeinput` | `InputEvent` | Si el texto escrito tiene algo que no sea dígito, hace `preventDefault` y avisa |
| `@HostListener` | `paste` | `ClipboardEvent` | Igual al pegar. Lee el portapapeles porque no todos los navegadores envían el texto pegado en `beforeinput` |

---

## Mensajes

| Si el texto tiene… | Título del aviso |
|---|---|
| Puntos o comas | "No uses puntos ni comas" |
| Espacios | "No uses espacios" |
| Cualquier otra cosa | "Solo se permiten números" |

Todos con el texto "Escribe 1500000, no 1.500.000".

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-10-14 | Yadilton Torres | Directiva inicial, compartida por la vista y el modal. 4 pruebas. |

---

## Observaciones

- No usar `(beforeinput)="condicion && metodo()"` para activarla a medias: cuando la condición es `false` la expresión devuelve `false` y Angular hace `preventDefault`, así que el campo no deja escribir nada. Para eso está `[appCampoSoloNumeros]="condicion"`.
- Los inputs usan `type="text"` con `inputmode="numeric"` (no `type="number"`), para poder ver y corregir valores viejos mal guardados como `1.500.000`.
