## Autor: Iker Acevedo
Fecha creación: 2026-09-16
Estado: preprod
Tipo: componente

# Trayectos

**Selector:** `app-trayectos`
**Ubicación:** `src/app/views/director-de-operaciones/costos-transportadora/components/trayectos`

---

## ¿Qué hace?

Gestiona el catálogo de códigos de ruta que una transportadora acepta (`URBANO`, `NACIONAL`, etc.) — la lista blanca que [`ValidadorTrayecto`](../../../../../backend/lambdas-dotnet/lambdas/ApiLambdaLiquidacionesLogighoAOT/motor-dinamico/validacion.md#validadortrayecto) usa para decidir si un pedido se puede tarifar tal cual, o si hace falta caer al trayecto por defecto.

Persiste en `TrayectosTransportadora`, vía el `TransportadoraActivaService`.

---

## Validaciones al guardar

- Código y Nombre Visible obligatorios.
- Orden entero ≥ 1.
- **Código único** entre los trayectos ya configurados (excluyendo el que se está editando) — el código es la clave real que usa el motor de cálculo; un duplicado lo dejaría ambiguo sobre cuál tarifa aplicar.

El código se fuerza a mayúsculas mientras se escribe, para que nunca dependa de cómo lo tipeó cada usuario.

---

## Marcar un trayecto como default

Un trayecto no "sabe" si es el default de la transportadora — ese dato vive en `ConfiguracionTransportadora.trayectoPorDefecto`, no en el documento del propio trayecto. Mezclarlo acá duplicaría la fuente de verdad y podría desincronizarse (dos lugares diciendo cosas distintas sobre cuál es el default). `marcarComoDefecto()` actualiza la Configuración General, no el trayecto.

---

## Eliminar un trayecto que es el default actual

El modal de confirmación cambia su texto explícitamente en este caso: avisa que, si se elimina, los pedidos sin trayecto reconocible van a auditoría hasta que se configure otro default — en vez de un genérico "esta acción no se puede deshacer" que no comunicaría la consecuencia real.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-09-16 | Iker Acevedo | Documentación inicial. |
