---

## Autor: Yadilton Torres

Fecha creacion: 2026-11-11  
Estado: activo  
Tipo: componente compartido

# Componente: Lista de opciones

**Selector:** `app-lista-opciones`  
**Ubicación:** `src/app/shared/components/lista-opciones/`  
**Dónde se usa:** [Panel comercial](../views/comercial/panel-comercial/panel-comercial.md), [Seguimiento de clientes](../views/comercial/seguimiento-clientes/seguimiento-clientes.md), [Mi perfil](../views/tienda/perfil/perfil.md)

---

## ¿Qué hace?

Lista desplegable con el estilo de la página. Reemplaza al `<select>` nativo, que no se puede personalizar.

```html
<app-lista-opciones etiqueta="Tipo de negocio" icono="fa-tags"
  [opciones]="opcionesTipo" [valor]="filtroTipo"
  (valorChange)="filtroTipo = $event; filtrar()"></app-lista-opciones>
```

| Entrada / salida | Tipo | Descripción |
| --- | --- | --- |
| `opciones` | `OpcionLista[]` | `{ valor, etiqueta, cantidad? }`. `cantidad` se muestra a la derecha |
| `valor` | `string` | Valor elegido |
| `etiqueta` | `string` | Texto cuando no hay valor y `aria-label` |
| `icono` | `string` | Clase Font Awesome del icono de la izquierda |
| `siempreAbajo` | `boolean` | En formularios (modales): abre siempre hacia abajo; si no cabe, el contenedor con scroll baja lo necesario. Por defecto `false` |
| `valorChange` | `string` | Se eligió otra opción |

---

## Comportamiento

- **Buscador:** con más de 8 opciones aparece un buscador (sin tildes). Busca en **todas** las opciones, no solo en las pintadas.
- **Listas largas:** se pintan por tandas de `TANDA_OPCIONES` (60) y se agregan más al bajar con el scroll. Si ya hay una opción elegida más abajo, o se baja con el teclado, se pinta hasta ella. Lo filtrado se guarda mientras no cambien las opciones ni el texto. Antes la lista de ciudades (más de mil) pintaba todo al abrir y tardaba.
- **Hacia dónde abre:** hacia abajo o hacia arriba según el espacio del contenedor con scroll más cercano (con `siempreAbajo`, siempre hacia abajo), y hacia la izquierda si a la derecha no cabe. El alto de la lista se ajusta para que quepa completa (mínimo 120 px).
- **Solo una abierta:** al abrirse avisa con el evento `logi-desplegable-abierto` (`avisarDesplegableAbierto`) y las demás se cierran. El [Filtro de fechas](filtro-fechas.md) usa el mismo evento.
- **Se cierra al presionar fuera** (`document:pointerdown`). Se usa `pointerdown` y no `click` porque los paneles laterales detienen el clic para no cerrarse, y la lista no se enteraba.
- **Teclado:** ↑ ↓ para moverse, Enter elige, Esc cierra y devuelve el foco al botón.
- **Filtro activo:** si el valor es distinto de la primera opción ("Todos"), el botón se resalta en azul (`lista--con-valor`). Sobre fondos azules (por ejemplo el pie de las tablas) la pantalla fuerza el botón blanco para que no desaparezca.

---

## Historial de cambios

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-11-11 | Yadilton Torres | Abre a la izquierda si no cabe; solo una abierta a la vez; se cierra con `pointerdown` fuera |
| 2026-11-19 | Yadilton Torres | Entrada `siempreAbajo` para modales; listas largas por tandas para que abran rápido. Pruebas en `lista-opciones.component.spec.ts` |
