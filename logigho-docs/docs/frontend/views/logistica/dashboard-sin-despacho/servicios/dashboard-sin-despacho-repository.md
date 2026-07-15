## Conexión al servidor: DashboardSinDespachoRepository

**Ubicación:** `src/app/views/logistica/dashboard-sin-despacho/workers/dashboard-sin-despacho.repository.ts`  
**Acceso:** Usado internamente por `DashboardSinDespachoComponent`

---

### ¿Qué hace? (para el usuario)

Este servicio es el encargado de pedir los datos al servidor. Busca los pedidos y las tiendas que necesita el módulo, pero lo hace de forma controlada para no traer más información de la necesaria.

---

### Qué información trae

| Consulta | Qué trae |
|---|---|
| Pedidos | Los pedidos de los últimos 4 meses (contados desde el día 1 del mes hasta hoy), de las tiendas que tiene asignadas el usuario |
| Tiendas | El catálogo completo de tiendas asignadas al usuario, con su ecosistema y si maneja fulfillment |

Ambas consultas llegan comprimidas desde el servidor para que la transferencia sea más liviana, y se descomprimen después en segundo plano (ver la documentación del `data.worker`).

---

### Por qué se limita a los últimos 4 meses

Traer todo el historial de pedidos sería mucho más lento y pesado de lo necesario para un tablero de seguimiento reciente. Por eso, cada vez que se abre el módulo, solo se pide el rango de fechas que va desde el primer día del mes de hace 4 meses hasta el día de hoy. Por ejemplo, si hoy es cualquier día de julio, se traen los pedidos desde el 1 de marzo en adelante.

---

### Preparado para una futura mejora

El equipo está probando una nueva capacidad del servidor que permitiría pedir los resultados **ya calculados** (por ejemplo, "cuántos pedidos hay por mes y transportadora") en vez de traer todos los pedidos y calcularlo en el navegador. Esto sería mucho más rápido si la cantidad de pedidos crece bastante con el tiempo.

Por ahora esa capacidad todavía está en pruebas y no se usa en el módulo — pero ya se dejó preparado el código para poder activarla en el futuro cambiando solo una configuración interna, sin tener que rediseñar el resto del módulo.

---

### Historial de cambios

| Fecha | Cambio |
|---|---|
| 2026-07-10 | Creación del servicio, con límite de 4 meses en la consulta de pedidos y preparación para una futura forma de traer resultados ya calculados desde el servidor |

---

### Observaciones

- Si el usuario tiene acceso a "Todas" las tiendas, no se envía ningún filtro de tienda en la consulta — el servidor entiende que debe devolver todo lo que corresponda.
- Aunque hoy el catálogo de tiendas es pequeño (unos cientos de registros), la consulta está preparada para seguir funcionando aunque ese catálogo crezca y empiece a llegar en varias páginas.
