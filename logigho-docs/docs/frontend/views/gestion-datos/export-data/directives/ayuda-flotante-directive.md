---

## Autor: Yadilton Torres
Fecha creacion: 2026-10-21
Estado: desarrollo

# Directiva: AyudaFlotanteDirective

**Selector:** `[appAyuda]`  
**Ubicación:** `src/app/views/export-data/directives/ayuda-flotante.directive.ts`  
**Tipo:** `standalone`

---

## ¿Qué hace?

Muestra una ayuda con diseño propio al pasar el mouse por un elemento o al llegar a él con el teclado. Reemplaza el `title` del navegador, que no se puede diseñar y tarda en salir. La usa la pantalla de [Exportación de datos](../export-data.md).

La ayuda se pinta en el `body` para que no la corten los contenedores con `overflow`.

---

## Uso

```html
<!-- Un renglón -->
<button [appAyuda]="'Quitar este filtro'">✕</button>

<!-- Dos renglones: el salto de línea la parte; el primero sale resaltado -->
<button [appAyuda]="ayudaRango('este_mes')">Este mes</button>

<!-- Vacío o null: no se muestra -->
<span [appAyuda]="filtro.valor.length > 40 ? filtro.valor : ''">...</span>
```

---

## Entradas

| Entrada | Tipo | Descripción |
| --- | --- | --- |
| `appAyuda` | `string \| null \| undefined` | Texto de la ayuda. Un `\n` la parte en dos renglones. Vacío = no se muestra |

---

## Comportamiento

| Acción | Resultado |
| --- | --- |
| Pasar el mouse | Aparece después de **250 ms** (`ESPERA_MS`), para que no salte al pasar de largo |
| Llegar con el teclado (`Tab`) | Aparece de una vez (solo con `:focus-visible`, no al hacer clic) |
| Salir con el mouse, perder el foco, hacer clic, `Escape` o mover la página | Se cierra |
| Pantalla táctil (`hover: none`) | No aparece, porque quedaría pegada al tocar |

**Posición:** arriba y centrada sobre el elemento; si no cabe arriba va abajo. Se corre para no salirse de los lados (8px de margen) y la flecha sigue apuntando al centro del elemento (variable CSS `--ayuda-flecha`).

**Accesibilidad:** la ayuda tiene `role="tooltip"` y un `id` único; mientras se ve, el elemento lleva `aria-describedby` con ese `id`.

---

## Dónde se usa en Exportación

| Elemento | Texto |
| --- | --- |
| Botones de rango de fecha | `ayudaRango(rango)`: descripción del rango y sus fechas exactas (ej. "Del día 1 de este mes hasta hoy" + `01/10/2026 – 21/10/2026`) |
| Columnas | Nombre de la columna (se ve completo aunque en la tarjeta salga cortado) |
| Nota "uno o varios separados por coma" | "También puedes pegar una columna de Excel" |
| Quitar filtro | "Quitar este filtro" |
| Valores de los filtros aplicados y del resumen | El valor completo (ej. todas las guías cuando el filtro dice "3 guías") |
| Botón Exportar | "Arma el archivo en tu navegador" + "Se descarga en una pestaña nueva; ideal para la mayoría de exportaciones" |

---

## Estilos

Globales en `src/scss/_ayuda-flotante.scss` (importado en `styles.scss`), porque la ayuda vive en el `body`, fuera del componente. Tarjeta blanca y suave con una línea azul arriba, primer renglón en azul oscuro, flecha blanca y aparición con transición (`ayuda-flotante--visible`). Ancho máximo 260px.

---

## Pruebas

`ayuda-flotante.directive.spec.ts`: 5 pruebas (aparece en el body ligada al elemento, se va con mouse/clic/Escape, no aparece si se pasa de largo, sin texto no muestra nada, no aparece en pantallas táctiles).

---

## Historial de cambios

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-10-21 | Yadilton Torres | Creación de la directiva; reemplaza todos los `title` de Exportación de datos |
| 2026-10-21 | Yadilton Torres | Diseño en tarjeta blanca (antes degradado oscuro, se veía muy fuerte) |

---

## Observaciones

- **Está en la carpeta de Exportación** porque hoy solo la usa esa pantalla. Si otra vista la necesita, conviene moverla a `components/` o `shared/`.
