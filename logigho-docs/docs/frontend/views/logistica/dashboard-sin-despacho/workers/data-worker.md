## Hilo de descarga: data.worker

**Ubicación:** `src/app/views/logistica/dashboard-sin-despacho/workers/data.worker.ts`  
**Acceso:** Usado internamente por `DashboardSinDespachoComponent`

---

### ¿Qué hace? (para el usuario)

Este proceso recibe la información del servidor, la descomprime y la deja lista para que se vea en la pantalla. Lo hace en segundo plano para que la interfaz no se quede pegada mientras carga datos.

---

### Qué hace paso a paso

1. Recibe el catálogo completo de tiendas y lo guarda internamente, para poder completar los datos de ecosistema y fulfillment de cada pedido.
2. Recibe cada página de pedidos que llega del servidor, la descomprime y convierte cada documento a un formato limpio y estandarizado.
3. Por cada pedido, busca su tienda correspondiente en el catálogo ya guardado y le agrega el ecosistema y el fulfillment.
4. Descarta automáticamente los pedidos anulados y los que tienen la transportadora mal escrita.
5. Le avisa a la pantalla principal cada vez que termina de procesar un lote, para que las tablas se actualicen con la nueva información.

---

### Por qué el orden importa

El catálogo de tiendas debe llegar **antes** que los pedidos. Si un pedido se procesara antes de que su tienda esté en el catálogo, ese pedido quedaría con el ecosistema y el fulfillment vacíos, sin poder completarse después. Por eso, la pantalla principal espera la confirmación de que el catálogo ya se aplicó por completo antes de empezar a enviar pedidos a este proceso.

---

### Flujo principal

```
Llega el catálogo de tiendas
  └─► se descomprime y se guarda en memoria del proceso
  └─► se confirma a la pantalla que ya está listo

Llega una página de pedidos
  └─► se descomprime
  └─► por cada pedido:
        - se revisa si su estado corresponde a un pedido anulado (se descarta)
        - se revisa si la transportadora está bien escrita (si no, se descarta)
        - se completa con el ecosistema y fulfillment de su tienda
        - se calcula su categoría de Estado Logi
  └─► se entrega la lista de pedidos ya limpios a la pantalla principal
```

---

### Historial de cambios

| Fecha | Cambio |
|---|---|
| 2026-07-10 | Creación del proceso de descarga y limpieza de pedidos, con cruce automático contra el catálogo de tiendas |

---

### Observaciones

- Este proceso corre completamente separado de la pantalla principal (en lo que técnicamente se llama un "Web Worker"), así que aunque esté procesando miles de registros, el usuario no nota ninguna lentitud al interactuar con el módulo.
- Si el teléfono o el número de guía de un pedido vienen en un formato especial de número muy largo, el proceso lo convierte automáticamente a texto para no perder ningún dígito.
