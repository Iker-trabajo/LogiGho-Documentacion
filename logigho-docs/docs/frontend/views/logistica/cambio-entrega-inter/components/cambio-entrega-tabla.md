---

## Autor:
Fecha creacion: 2026-08-20
Estado: produccion
Tipo: componente

# Componente: CambioEntregaTablaComponent

**Selector:** `app-cambio-entrega-tabla`
**Ubicación:** `src/app/views/logistica/cambio-entrega-inter/components/cambio-entrega-tabla/`

---

## ¿Qué hace?

Pinta la lista de guías auditadas — tabla completa en desktop, cards resumidas en mobile — más el footer de paginación compartido por las dos vistas. Es un componente "tonto": recibe la página ya recortada por el padre (`itemsPaginados`), no decide qué mostrar, solo cómo mostrarlo.

---

## Decoradores

| Decorador | Nombre | Tipo | Descripción |
| --------- | ------ | ---- | ----------- |
| `@Input` | `items` | `CambioEntregaInter[]` | Página ya recortada — el padre decide qué mostrar |
| `@Input` | `currentPage`, `totalPaginas`, `pageSize`, `pageSizeOptions`, `totalRegistros` | — | Estado de paginación, vive en el padre |
| `@Input` | `guardandoIds` | `Set<string>` | `_id` de guías con un `PUT` de gestión en curso — deshabilita el switch de esa fila mientras tanto |
| `@Output` | `verDetalle` | `EventEmitter<CambioEntregaInter>` | Clic en "Ver detalle" |
| `@Output` | `paginaChange` | `EventEmitter<number>` | Cambio de página |
| `@Output` | `pageSizeChange` | `EventEmitter<number>` | Cambio de tamaño de página |
| `@Output` | `cambiarGestion` | `EventEmitter<{item, nuevoEstado}>` | Switch de la columna "Gestión" |

---

## Columnas (vista desktop)

17 columnas con scroll horizontal, columna "Acciones" fija a la derecha (`position: sticky`). Dos columnas de estado que se parecen pero **no son lo mismo**:

| Columna | Fuente | Nota |
| ------- | ------ | ---- |
| `Estado` | `item.Estado` | Siempre `"Reclame en oficina"` en esta vista — congelado desde la detección |
| `Estado Final` | `item.EstadoFinal` | Desenlace real y actual: Entrega (verde) / Devolución (roja) / Pendiente (azul) |
| `Fecha Creación Pedido` | `item.FechaCreacionPedido` | Antes de que Inter tocara el pedido |
| `Fecha Reclame Oficina (Inter)` | `item.FechaEstadoOficina` | Timestamp que reportó Inter, no el nuestro |

---

## Propiedades clave

| Propiedad | Tipo | Descripción |
| --------- | ---- | ----------- |
| `formatearMoneda(valor: number)` | `string` | `$ 120.000` — separador de miles armado a mano (`replace` con regex), no `toLocaleString('es-CO')`: algunos navegadores no traen los datos ICU completos de esa locale y el separador simplemente no aparece |
| `formatearFecha(fecha: string)` | `string` | `dd de mmm de yyyy` |

---

## Flujo principal

```
Switch de gestión (columna "Gestión")
  -> onToggleGestion(item)
  -> invierte EstadoGestion (Pendiente <-> Gestionada)
  -> cambiarGestion.emit({item, nuevoEstado})
     (el padre hace el PUT y actualiza el signal en memoria)
```

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-31 | Iker Acevedo | Tabla inicial: columnas, cards mobile, paginación. |
| 2026-08-18 | Iker Acevedo | Columna/badge de `Estado Final`. `TotalRecaudo` pasa a `number` — `formatearMoneda` reescrito sin negrilla y con separador manual. |

---

## Observaciones

- `formatearMoneda` está duplicado en `cambio-entrega-detalle-modal` (mismo criterio) — no hay un pipe/servicio compartido para esto en el módulo todavía.
- La columna "Acciones" fija (`sticky`) tiene su propia sombra a la izquierda para marcar dónde empieza el contenido que sí se desplaza, evitando que se vea "flotando" sobre el resto de la tabla.
