# Módulo: Dashboard Sin Despacho

---

## Autor: Adalberto González
Fecha creación: 2026-07-10  
Estado: desarrollo  
Tipo: módulo (1 vista + modelos, tienda de datos y hilos de procesamiento en segundo plano)

---

## Índice

1. [Vista: DashboardSinDespachoComponent](#1-vista-dashboardsindespachocomponent)
2. [Modelos: contratos de datos del módulo](modelos/sin-despacho-models.md)
3. [Tienda de datos: DashboardSinDespachoStore](modelos/dashboard-sin-despacho-store.md)
4. [Conexión al servidor: DashboardSinDespachoRepository](servicios/dashboard-sin-despacho-repository.md)
5. [Hilo de descarga: data.worker](workers/data-worker.md)
6. [Hilo de cálculo: agregacion.worker](workers/agregacion-worker.md)

---

## 1. Vista: DashboardSinDespachoComponent

**Selector:** `app-dashboard-sin-despacho`  
**Ubicación:** `src/app/views/logistica/dashboard-sin-despacho/dashboard-sin-despacho.component.ts`  
**Acceso:** Logística → Dashboard Sin Despacho

---

### ¿Qué hace? (para el usuario)

Esta pantalla sirve para ver de forma rápida qué pedidos todavía no han sido despachados y también mirar lo reciente que ya pasó. Al abrirla, el sistema trae los pedidos de los últimos 4 meses y los organiza para que sea más fácil entenderlos.

Puedes ver:

- **3 resúmenes principales** con el total de pedidos, los que aún están sin despachar y el dinero total vendido.
- **Una tabla por mes y transportadora** para comparar cómo van los pedidos en cada periodo.
- **Un ranking de tiendas** para ver qué tiendas tienen más pedidos o más problemas de despacho.
- **Una tabla detallada** con cada pedido individual, incluyendo teléfono, fecha, guía, tienda, estado y más.
- **Filtros** para buscar por ciudad, mes, tienda, transportadora, ecosistema, estado del pedido y tipo de fulfillment.
- **Selecciones combinadas**: si haces clic en un mes, una tienda o una transportadora, las demás tablas se actualizan para mostrar solo lo relacionado.
- **Exportar a Excel** desde cada tabla para guardar o compartir la información.

---

### Ruta

```
logistica/dashboard-sin-despacho
```

---

### ¿De dónde vienen los datos?

El módulo consulta dos fuentes:

1. **Pedidos (colección PedidosInter):** se traen solo los últimos 4 meses, contados desde el día 1 del mes hasta hoy. Esto evita descargar años de historial innecesariamente.
2. **Tiendas (colección Tienda):** un catálogo con la información de cada tienda (a qué ecosistema pertenece y si maneja fulfillment o no). Esta información no viene en el documento del pedido, así que el sistema "cruza" cada pedido con su tienda para completar esos datos.

Antes de mostrar cualquier pedido, el sistema descarta automáticamente:

- Los pedidos **anulados** (no se cuentan en ninguna parte del tablero).
- Los pedidos cuya transportadora esté mal escrita (por ejemplo, con un espacio de más), para que esas filas no aparezcan como una columna repetida y confusa en la tabla.

---

### Los "Estados Logi"

Las transportadoras reportan el estado de un pedido con textos muy variados y específicos (por ejemplo "EN REPARTO EN MEDELLIN", "EN NOVEDAD POR CALI", "SOLUCIONADA EN MALLA EN BOGOTA"). El sistema agrupa automáticamente todos esos textos en 7 categorías simples para que sea fácil de entender y filtrar:

| Categoría | Qué significa |
|---|---|
| Generada | El pedido se acaba de crear |
| Novedad | Hay un problema o inconveniente con la entrega |
| Transito | El pedido está en camino |
| Entregada | El pedido ya llegó a su destino |
| Devolución | El pedido está siendo devuelto |
| Devolución Recibida | La devolución ya fue recibida de vuelta |
| Indemización | El pedido entró en proceso de indemnización |

Si en algún momento una transportadora reporta un texto nuevo que el sistema no reconoce, ese pedido no se pierde: queda marcado como "Sin clasificar" y se registra un aviso para que se pueda agregar esa categoría más adelante.

---

### Indicadores (KPIs)

| Indicador | Qué muestra |
|---|---|
| Total pedidos | Cantidad de pedidos que cumplen los filtros y selecciones activas |
| Sin despacho | Cuántos de esos pedidos todavía no se han despachado |
| Ventas totales | Suma del dinero recaudado en esos pedidos |

---

### Servicios y endpoints

| Servicio | Uso |
|---|---|
| `DashboardSinDespachoRepository` | Trae las páginas de pedidos y el catálogo de tiendas desde el servidor |
| `DashboardSinDespachoStore` | Guarda en memoria todo lo que ya se descargó y lo que el usuario ve en pantalla |

**Patrón de URL:**
```
metodoGenerico?coleccion=PedidosInter&fechasFiltro=<rango de 4 meses>&Tienda=<tiendas del usuario>&page=<N>
```

> Si el usuario tiene acceso a "Todas" las tiendas, se omite el filtro de tienda para traer todos los registros.

---

### Flujo principal

```
Se abre la pantalla
  └─► Primero se descarga el catálogo completo de Tiendas
        (se espera a que termine antes de seguir, así ningún
         pedido queda sin su información de ecosistema/fulfillment)
  └─► Luego se descargan los pedidos de los últimos 4 meses,
        página por página, mostrando resultados apenas van llegando
  └─► Cada vez que llega una página nueva, se recalculan
        automáticamente las 3 tablas y los indicadores

Usuario hace clic en un mes, día, tienda, transportadora o pedido
  └─► Esa selección se guarda y se combina con las demás
        selecciones activas (no las reemplaza)
  └─► Las demás tablas se recalculan mostrando solo lo relacionado
        con la selección (la tabla donde se hizo clic no se
        autolimita, para poder seguir comparando)

Usuario exporta una tabla a Excel
  └─► Se genera un archivo .xlsx con exactamente lo que esa
        tabla está mostrando en ese momento (filtros y
        selecciones incluidos)
```

---

### Historial de cambios

| Fecha | Cambio |
|---|---|
| 2026-07-10 | Creación del módulo: tabla de pedidos por mes y transportadora, ranking de tiendas, detalle de pedidos, filtros combinables, exportación a Excel independiente por tabla, selección cruzada combinable entre mes/día/tienda/transportadora/pedido |

---

### Observaciones

- El módulo no bloquea la pantalla mientras carga: los datos aparecen progresivamente a medida que van llegando del servidor, en vez de esperar a que se descargue todo antes de mostrar algo.
- Todo el trabajo pesado (descomprimir la información que envía el servidor y calcular las tablas) ocurre en segundo plano, sin congelar la pantalla mientras el usuario interactúa.
- El equipo está evaluando una segunda forma de traer los datos, en la que el servidor entregaría los resultados ya calculados en vez de que el navegador tenga que calcularlos — esto haría el módulo mucho más rápido si la cantidad de pedidos crece mucho con el tiempo. Por ahora sigue funcionando con el método actual (descargar y calcular en el navegador) porque esa nueva forma todavía está en pruebas.
