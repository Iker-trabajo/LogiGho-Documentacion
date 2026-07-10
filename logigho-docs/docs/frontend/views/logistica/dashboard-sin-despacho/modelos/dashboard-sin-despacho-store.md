## Tienda de datos: DashboardSinDespachoStore

**Ubicación:** `src/app/views/logistica/dashboard-sin-despacho/workers/dashboard-sin-despacho.store.ts`  
**Acceso:** Usado internamente por `DashboardSinDespachoComponent`

---

### ¿Qué hace? (para el usuario)

Este archivo funciona como la memoria del módulo. Guarda los pedidos que ya llegaron, los filtros que eligió la persona y los resultados que se muestran en pantalla, para que todo quede ordenado y actualizado.

---

### Qué información guarda

| Información | Para qué sirve |
|---|---|
| Todos los pedidos descargados | La base de datos completa que ya se trajo del servidor, sin filtrar |
| Filtros activos | Ciudad, mes, tienda, transportadora, ecosistema, estado logístico y fulfillment seleccionados por el usuario |
| Selección cruzada | El mes, día, tienda, transportadora o pedido específico en el que el usuario hizo clic |
| Página actual | En qué página está la tabla de detalle |
| Resultados calculados | La tabla de pedidos por mes, el ranking de tiendas, los indicadores y el detalle paginado, listos para mostrarse |
| Estado de carga | Si todavía se están descargando datos o si ya se calculó todo |

---

### Cómo evita duplicados

Cada vez que llega un lote nuevo de pedidos desde el servidor, el sistema revisa cuáles ya estaban guardados (usando su identificador único) y solo agrega los que son realmente nuevos. Esto evita que un mismo pedido aparezca dos veces si, por alguna razón, el servidor lo vuelve a enviar.

---

### Cómo se combinan las selecciones

Cuando el usuario hace clic en un mes, día, tienda, transportadora o pedido, el sistema no reemplaza la selección anterior — la agrega. Así, se pueden tener varias selecciones activas al mismo tiempo (por ejemplo: un día + una tienda), y las demás tablas del tablero muestran la intersección exacta de esas selecciones. Si el usuario vuelve a hacer clic en lo mismo, esa selección se quita sin afectar las demás.

---

### Flujo principal

```
El módulo empieza a cargar
  └─► setLoading(true) → aparece el indicador de "cargando"

Llega un lote de pedidos nuevo
  └─► appendLote() → se agregan solo los pedidos que no estaban antes

El usuario cambia un filtro o hace clic en una selección
  └─► setFiltros() / setSeleccion() → se guarda el cambio
        y se reinicia la página a la 1

Terminó de calcularse una tabla en segundo plano
  └─► setAggResult() → se guardan los resultados listos
        para mostrarse en pantalla

El usuario sale del módulo o lo recarga
  └─► reset() → se limpia toda la memoria del módulo
```

---

### Historial de cambios

| Fecha | Cambio |
|---|---|
| 2026-07-10 | Creación de la tienda de datos, incluyendo el soporte para selecciones combinables entre tablas |

---

### Observaciones

- Este componente nunca hace peticiones al servidor por sí mismo — solo guarda y organiza la información que otras partes del módulo le entregan.
- Cuando cambia un filtro o una selección, la página de la tabla de detalle siempre vuelve a la primera automáticamente, para que el usuario no se quede viendo una página vacía si el nuevo filtro tiene menos resultados que antes.
