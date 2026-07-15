## Hilo de cálculo: agregacion.worker

**Ubicación:** `src/app/views/logistica/dashboard-sin-despacho/workers/agregacion.worker.ts`  
**Acceso:** Usado internamente por `DashboardSinDespachoComponent`

---

### ¿Qué hace? (para el usuario)

Este proceso se encarga de preparar las tablas y los números del tablero a partir de los pedidos que ya llegaron. Lo hace en segundo plano para que la pantalla responda rápido aunque haya mucha información.

---

### Cómo aplica los filtros y selecciones

Primero aplica los filtros generales (ciudad, mes, tienda, transportadora, ecosistema, estado logístico, fulfillment). Sobre ese resultado, aplica además las selecciones activas (el mes, día, tienda, transportadora o pedido en el que el usuario hizo clic), que se combinan todas entre sí.

Hay una regla especial: **la tabla donde el usuario hizo clic no se limita a sí misma.** Por ejemplo, si el usuario selecciona un mes en la tabla de pedidos por mes, esa misma tabla sigue mostrando todos los meses (para poder seguir comparando o cambiar de selección fácilmente), pero el ranking de tiendas, los indicadores y el detalle sí quedan filtrados a ese mes. Lo mismo aplica al revés: si se selecciona una tienda en el ranking, la tabla de pedidos por mes sí se filtra a esa tienda, pero el ranking sigue mostrando todas las tiendas.

---

### Cómo se calculan las opciones de los filtros

Cada filtro (por ejemplo, "Fulfillment") calcula sus opciones disponibles ignorando su propio filtro pero respetando todos los demás. Esto evita un problema común: si el usuario ya seleccionó "Sí" en Fulfillment, la opción "No" no debería desaparecer del listado — debe seguir visible para que el usuario pueda marcarla también si quiere ver ambas.

---

### Exportación a Excel

Cuando el usuario pide exportar una tabla, este proceso recalcula esa tabla específica pero sin la excepción mencionada arriba — es decir, si hay una selección de mes activa, el Excel de "pedidos por mes y transportadora" sí sale acotado exactamente a esa selección, aunque en pantalla la tabla siga mostrando todos los meses. Lo mismo ocurre con el detalle: se exportan **todos** los pedidos que cumplen los filtros, no solo los de la página que se está viendo en ese momento.

---

### Flujo principal

```
Cambia un filtro, una selección, o se pide exportar
  └─► se aplican los filtros generales sobre todos los pedidos
  └─► se aplican las selecciones activas (combinadas)
  └─► se calcula la tabla de pedidos por mes y transportadora
  └─► se calcula el ranking de tiendas
  └─► se calculan los indicadores superiores
  └─► se calculan las opciones disponibles de cada filtro
  └─► se entrega todo el resultado a la pantalla principal
        para que se muestre de inmediato
```

---

### Historial de cambios

| Fecha | Cambio |
|---|---|
| 2026-07-10 | Creación del cálculo de tablas e indicadores, con selección combinable entre mes, día, tienda, transportadora y pedido, y exportación acotada por tabla |

---

### Observaciones

- Los pedidos anulados y las transportadoras mal escritas ya vienen descartados desde el paso anterior (`data.worker`), así que este proceso nunca necesita volver a filtrarlos.
- El orden de las opciones del filtro "Estados Logi" no es alfabético — sigue el orden natural del proceso de un pedido (Generada, Novedad, Tránsito, Entregada, Devolución, Devolución Recibida, Indemización), para que sea más intuitivo de leer.
