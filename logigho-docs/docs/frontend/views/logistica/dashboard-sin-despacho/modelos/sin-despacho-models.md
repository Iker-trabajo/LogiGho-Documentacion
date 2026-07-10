## Modelos: contratos de datos del módulo

**Ubicación:** `src/app/views/logistica/dashboard-sin-despacho/models/sin-despacho.models.ts`  
**Acceso:** Usado internamente por todo el módulo Dashboard Sin Despacho

---

### ¿Qué hace? (para el usuario)

Este archivo ayuda a que todo el módulo hable el mismo idioma. Define cómo se guarda y se organiza la información de cada pedido, para que la pantalla, los cálculos y los filtros funcionen de forma consistente.

---

### Un pedido, antes y después de procesarse

**Como llega del servidor (crudo):** un documento de Mongo con muchos campos, algunos con formatos poco amigables — por ejemplo el teléfono puede venir como texto o como un tipo especial de número largo, y el total recaudado puede venir con separadores de miles que no siempre son consistentes.

**Como queda una vez procesado:** una fila limpia y lista para usar en toda la pantalla, con estos datos principales:

| Dato | Qué es |
|---|---|
| Guía | Número de guía del envío |
| Teléfono | Teléfono del cliente |
| Fecha de carga | Fecha en la que se registró el pedido |
| Tienda | Nombre de la tienda |
| Transportadora | Transportadora asignada |
| Estado | Estado crudo reportado por la transportadora |
| Estado Logi | La categoría simplificada del estado (ver tabla de las 7 categorías en la documentación de la vista) |
| Ciudad / Departamento | Ubicación de entrega |
| Ventas totales | Dinero recaudado en ese pedido |
| Ecosistema | A qué ecosistema pertenece la tienda (dato que viene del catálogo de Tiendas, no del pedido) |
| Fulfillment | Si la tienda maneja fulfillment o no (también viene del catálogo de Tiendas) |

---

### Cómo se agrupan los resultados

- **Pedidos por mes y transportadora:** una tabla que agrupa la cantidad de pedidos por cada mes, y dentro de cada mes por cada transportadora. Además, cada mes se puede abrir para ver el detalle día por día.
- **Ranking de tiendas:** una lista de tiendas con su cantidad de pedidos, cuántos están sin despachar y cuánto han vendido.
- **Filtros disponibles:** las opciones que se pueden seleccionar en cada filtro (ciudad, mes, tienda, transportadora, ecosistema, estado logístico, fulfillment) se calculan automáticamente según los datos que hay cargados en ese momento — así el usuario nunca ve una opción que no tenga ningún pedido asociado.

---

### La selección que combina varias tablas a la vez

Cuando el usuario hace clic en un mes, un día, una tienda, una transportadora o un pedido específico, el sistema recuerda todas esas selecciones al mismo tiempo (no solo la última). Por ejemplo: si el usuario primero selecciona un día y luego selecciona una tienda, el sistema entiende que quiere ver "los pedidos de esa tienda, en ese día" — no reemplaza una selección por la otra.

---

### Normalización de estados: cómo se decide la categoría de cada pedido

Como cada transportadora puede reportar cientos de textos de estado distintos (muchos de ellos con el nombre de una ciudad incluido, como "EN REPARTO EN MEDELLIN"), el sistema usa dos mecanismos para clasificarlos:

1. **Lista exacta:** los textos de estado más comunes y fijos (como "Creado", "Entregada", "Anulada") están mapeados uno por uno a su categoría correspondiente.
2. **Reglas por inicio de texto:** para los textos que cambian según la ciudad (por ejemplo "EN REPARTO EN..." o "EN NOVEDAD POR..."), el sistema revisa cómo empieza el texto y lo clasifica según ese patrón, sin necesidad de listar cada ciudad una por una.

Si un texto de estado no coincide con ninguna de las dos reglas, el pedido se clasifica como "Sin clasificar" y se guarda un aviso técnico (solo una vez por texto nuevo) para que el equipo pueda revisarlo y agregar esa categoría si hace falta.

---

### Pedidos que se descartan por completo

Antes de que un pedido entre a cualquier tabla o indicador del tablero, se descarta si:

- Su estado corresponde a la categoría **Cancelada** (pedidos anulados).
- Su transportadora está mal escrita — por ejemplo, con espacios de más o en minúsculas cuando debería estar en mayúsculas. Esto evita que aparezcan columnas duplicadas y confusas en la tabla de transportadoras (por ejemplo, "INTERRAPIDISIMO" y "INTERRAPIDISIMO " contando como dos transportadoras distintas).

---

### Historial de cambios

| Fecha | Cambio |
|---|---|
| 2026-07-10 | Creación de los modelos: contrato de pedido crudo y procesado, categorías de Estado Logi con lista exacta + reglas por prefijo, selección combinable entre tablas |

---

### Observaciones

- Cuando el estado de un pedido no coincide con ninguna categoría conocida, el sistema no lo oculta ni lo excluye — lo deja visible como "Sin clasificar" para que no se pierda información, y solo avisa una vez por cada texto nuevo (no satura la consola con avisos repetidos).
- El campo de ventas totales se limpia de forma defensiva: si un pedido trae un valor de dinero con un formato raro o corrupto, en vez de romper el cálculo total, ese pedido puntual se cuenta como $0 en vez de hacer fallar toda la suma.
