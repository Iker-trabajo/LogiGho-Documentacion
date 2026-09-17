## Autor: Iker Acevedo
Fecha creación: 2026-09-16
Estado: preprod
Tipo: componente

# Configuración General

**Selector:** `app-configuracion-general`
**Ubicación:** `src/app/views/director-de-operaciones/costos-transportadora/components/configuracion-general`

---

## ¿Qué hace?

Edita el documento `ConfiguracionLiquidacionTransportadora` de la transportadora activa: qué motor usa, qué política de peso, el tope de kilos, si aplica el máximo servicio/seguro, y el trayecto por defecto. Es el panel que decide si una transportadora liquida de verdad con el motor Dinámico o no.

---

## Patrón de borrador (`draft`)

El componente edita una copia local (`draft`) que arranca como clon de `store.configuracion()`. Solo se escribe de vuelta al store cuando el usuario presiona **Guardar** — así "Descartar" puede volver al último estado guardado sin lógica adicional, simplemente recargando el draft desde el store otra vez.

Todos los campos y reglas están alineados 1:1 con `ConfiguracionTransportadora.cs` del backend — no son supuestos de diseño de la pantalla.

---

## Validaciones de operación

Un panel muestra reglas reales de consistencia (copiadas de los comentarios del dominio backend), no un checklist decorativo:

| Validación | Bloqueante | Regla |
|---|---|---|
| Tope de kilos en rango | Sí | Entre 2 y 50 si la política es tabla; debe estar vacío si es porcentaje |
| Nombre en Pedidos configurado | Sí | Necesario para que el motor pueda asociar pedidos entrantes con esta configuración |
| Trayecto por defecto válido | No | Si no está configurado, los pedidos sin trayecto reconocible van directo a auditoría |

Si hay alguna validación **bloqueante** sin cumplir, el botón Guardar queda inhabilitado — motor Dinámico con datos inconsistentes es exactamente el escenario que puede hacer fallar una liquidación real, y se corta acá, antes de que ese dato llegue a la Lambda.

El tope físico de 50 kg es un límite de **negocio** (no hay ninguna tarifa real que lo necesite por encima de eso), por eso vive en el componente y no en el modelo de dominio del backend.

---

## Comportamientos ya resueltos, no obvios

- Cambiar la política de peso a "Porcentaje sobre Flete" limpia automáticamente el tope de kilos (`null`, porque esa política no lo usa); cambiar a "Tabla de Incrementos" lo inicializa en 5 si estaba vacío.
- El campo de tope de kilos se corrige en el momento (`onCambioTopeKilos`) si se pega un valor fuera de rango — el atributo `max` del `<input>` solo limita las flechitas del spinner, no lo que se puede pegar directo.
- El selector de "Trayecto por Defecto" se llena con los trayectos reales configurados en la pestaña Trayectos — ya no son 3 valores fijos ("Urbano", "Nacional", "Especial") como en una versión anterior; el backend acepta cualquier código que exista en la lista blanca.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-09-16 | Iker Acevedo | Documentación inicial. |

---

## Observaciones

- El motor y la política se muestran con su descripción completa bajo el select (`descripcionMotorSeleccionado`/`descripcionPoliticaSeleccionada`) — copiada literal del comentario de cada enum en `Enums.cs`, para que quien configura entienda la consecuencia de cada opción sin tener que preguntar.
