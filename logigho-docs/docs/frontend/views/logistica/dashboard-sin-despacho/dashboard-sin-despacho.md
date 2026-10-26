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
- **Una tabla por mes y transportadora** para comparar cómo van los pedidos en cada periodo. Los pedidos se ubican según su **fecha de creación**, y cada mes se puede abrir para ver el detalle día por día.
- **Varios días a la vez**: dentro de un mes abierto puedes marcar uno o varios días (ver [Selección de varios días](#seleccion-de-varios-dias)) y una fila naranja muestra la suma de lo marcado.
- **Un ranking de tiendas** para ver qué tiendas tienen más pedidos o más problemas de despacho.
- **Una tabla detallada** con cada pedido individual, incluyendo teléfono, fecha de creación, guía, tienda, estado y más.
- **Filtros** para buscar por ciudad, fecha de creación, tienda, transportadora, ecosistema, estado del pedido y tipo de fulfillment.
- **Selecciones combinadas**: si haces clic en un mes, una tienda o una transportadora, las demás tablas se actualizan para mostrar solo lo relacionado.
- **Un resumen de lo que estás viendo** ("Mostrando 312 de 144.724 pedidos") que se despliega con un botón para ver y quitar cada filtro o selección.
- **Botón Limpiar**, al lado de "Actualizar", que quita de una vez todos los filtros y selecciones. Solo aparece cuando hay algo que limpiar.
- **Exportar a Excel** desde cada tabla para guardar o compartir la información.
- **Una versión pensada para celular** (ver [En celular](#en-celular)).

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

### Filtros

Cada filtro abre una lista con buscador y casillas para elegir uno o varios valores.

- El selector siempre ocupa **una sola línea**: muestra el primer valor elegido y un "+N" si hay más. Pasando el mouse se ven todos.
- Al abrir la lista, las opciones **ya marcadas aparecen primero**, y mientras vas marcando la lista no cambia de orden.
- Filtros como Tienda o Ciudad pueden tener miles de opciones. Para que la lista abra rápido se muestran **máximo 150** y abajo aparece "Mostrando 150 de N · escribe para buscar el resto". Buscando se encuentran todas.
- Dar varios clics seguidos no selecciona el texto de la lista.

---

### Selección de varios días

Dentro de la tabla por mes y transportadora, al abrir un mes (botón **+**) aparecen sus días. Sobre esas filas:

| Acción | Qué hace |
|---|---|
| Clic | Deja marcado solo ese día. Si ya era el único marcado, lo quita |
| Ctrl + clic | Agrega o quita ese día sin tocar los demás |
| Shift + clic | Marca todos los días entre el último que se cliqueó y este |
| Ctrl + Shift + clic | Suma ese rango a lo que ya estaba marcado |

En celular no hay Ctrl ni Shift, así que **cada toque suma o quita un día**.

Mientras haya días marcados aparece la fila naranja **"Seleccionados (N días)"**, encima del Total, con la suma por transportadora. Los indicadores, el ranking, el detalle y los Excel cuentan solo esos días.

Al marcar días se quita el mes que estuviera seleccionado, para que los indicadores no queden en cero al combinar días de un mes con otro mes distinto. Los meses abiertos se quedan abiertos mientras marcas, y aunque hagas clics muy seguidos siempre se calcula con la última selección.

---

### En celular

En pantallas de 768 px o menos la vista se reorganiza para que los datos se vean sin bajar tanto:

| Parte | Cómo se ve en celular |
|---|---|
| Encabezado | Título grande a todo el ancho y una fila de botones: **Filtros** (con el número de filtros activos), Limpiar, Actualizar y el tour, estos tres solo con ícono |
| Filtros | No ocupan pantalla: el botón **Filtros** los abre en una hoja que sube desde abajo, con el botón **Listo**. Cada lista de opciones también se abre como hoja |
| Resumen | Una sola línea con los contadores. Al abrirlo, los chips se deslizan de lado |
| Indicadores | Los tres en una fila |
| Tabla por mes | Tiene su propio scroll (hasta el 62 % del alto de la pantalla), con el encabezado arriba y las filas Seleccionados y Total abajo siempre visibles. La columna Mes/Día queda fija al deslizar de lado |
| Ranking | Cabe completo sin deslizar de lado; los nombres largos se cortan con "…" |
| Detalle | Cada pedido es una tarjeta compacta: guía con botón de copiar y valor, estado y fecha, tienda y transportadora, teléfono y departamento. Las observaciones aparecen con "Ver observaciones". Se muestran **5 pedidos por página** (50 en computador) |

El ancho de la vista es el mismo con o sin filtros: los números largos no ensanchan la página. Los estilos de celular están en `styles/dashboard-sin-despacho.mobile.scss`, que se carga después de `styles/dashboard-sin-despacho.scss` y solo cambia lo necesario.

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
  └─► La primera página se calcula y se muestra de inmediato; las
        siguientes se agrupan y se recalcula como máximo cada 300 ms,
        con un cálculo final al llegar la última página

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
| 2026-10-22 | Agrupación por fecha de creación; selección de varios días (clic, Ctrl, Shift) con fila de sumatoria; botón Limpiar; resumen plegable; listas de filtros más rápidas; carga con recálculo agrupado; versión para celular (hoja de filtros, tabla con scroll propio, tarjetas compactas, 5 pedidos por página). Corregido: los meses se cerraban solos al marcar un día y se perdía el último clic rápido |

---

### Observaciones

- El módulo no bloquea la pantalla mientras carga: los datos aparecen progresivamente a medida que van llegando del servidor, en vez de esperar a que se descargue todo antes de mostrar algo.
- La pantalla agrupa y muestra por **fecha de creación**, pero el rango de fechas que se le pide al servidor lo aplica el backend, y hoy lo compara contra la fecha de carga (`FechaCarga`). Para que el rango también sea por fecha de creación hay que cambiarlo en el backend.
- Todo el trabajo pesado (descomprimir la información que envía el servidor y calcular las tablas) ocurre en segundo plano, sin congelar la pantalla mientras el usuario interactúa.
- El equipo está evaluando una segunda forma de traer los datos, en la que el servidor entregaría los resultados ya calculados en vez de que el navegador tenga que calcularlos — esto haría el módulo mucho más rápido si la cantidad de pedidos crece mucho con el tiempo. Por ahora sigue funcionando con el método actual (descargar y calcular en el navegador) porque esa nueva forma todavía está en pruebas.
