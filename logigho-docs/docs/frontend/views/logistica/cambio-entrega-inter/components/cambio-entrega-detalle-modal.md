---

## Autor:
Fecha creacion: 2026-08-20
Estado: produccion
Tipo: componente

# Componente: CambioEntregaDetalleModalComponent

**Selector:** `app-cambio-entrega-detalle-modal`
**Ubicación:** `src/app/views/logistica/cambio-entrega-inter/components/cambio-entrega-detalle-modal/`

---

## ¿Qué hace?

Modal de detalle de una guía: datos del pedido, timeline del historial completo (incluido el desenlace final si ya se resolvió), acciones de contacto (WhatsApp, copiar guía/teléfono) y el deep-link a la plataforma de pedidos para gestionar la guía.

---

## Decoradores

| Decorador | Nombre | Tipo | Descripción |
| --------- | ------ | ---- | ----------- |
| `@Input` | `isOpen` | `boolean` | Abre/cierra el modal |
| `@Input` | `item` | `CambioEntregaInter \| null` | Guía a mostrar |
| `@Input` | `guardando` | `boolean` | Deshabilita el dropdown de gestión mientras guarda |
| `@Output` | `cerrar` | `EventEmitter<void>` | Cierre (X, clic afuera, Escape, o después de "Consultar en plataforma") |
| `@Output` | `gestionar` | `EventEmitter<EstadoGestion>` | Cambio de estado de gestión desde el dropdown del modal |

---

## Timeline (historial)

Ordenado del más reciente al más antiguo. El primer ítem (desenlace) es **condicional** — solo aparece si `item.EstadoFinal` existe:

```
[si EstadoFinal existe] "Se entregó" / "Se devolvió"   -> FechaResolucion
"Reclame en oficina"                                    -> FechaEstadoOficina (timestamp de Inter)
"Primera detección de auditoría"                         -> FechaPrimeraDeteccion
[si difiere] "Última vez confirmado en oficina"          -> FechaUltimaDeteccion
"Pedido creado"                                          -> FechaCreacionPedido
```

---

## Flujo principal

```
irAPlataforma()
  -> cerrar.emit()
  -> router.navigate(['/app/tienda/pedidos'])
     (Router.navigate(), no routerLink en un <a> — un <a> con routerLink
      sacaba al usuario de la SPA por completo)

whatsappUrl(telefono)
  -> https://api.whatsapp.com/send/?phone=57{soloDigitos}&text&type=phone_number
```

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-31 | Iker Acevedo | Modal inicial: datos del pedido, historial, dropdown de gestión. |
| 2026-08-15 | Iker Acevedo | Botón "Consultar en plataforma" (con búsqueda automática por deep-link — luego revertido a solo navegar, ver observaciones), WhatsApp, bloqueo de scroll de fondo mientras el modal está abierto. |
| 2026-08-18 | Iker Acevedo | Paso de "desenlace" en el timeline (`EstadoFinal`/`FechaResolucion`), condicional. `formatearMoneda` reescrito para `TotalRecaudo: number`. |

---

## Observaciones

- **Deep-link revertido:** la primera versión de "Consultar en plataforma" navegaba a `/app/tienda/pedidos?campo=Numeropreenvio&valor={guia}` y el módulo de pedidos auto-buscaba la guía. Se quitó esa funcionalidad por pedido explícito — el botón hoy solo navega al módulo, sin autobúsqueda ni query params; el usuario busca la guía a mano ahí. La lógica de auto-búsqueda que se había agregado en `pedidos.component.ts` (fuera de este módulo) también se revirtió — no queda rastro de ese deep-link en ningún lado.
- `formatearMoneda` está duplicado con `cambio-entrega-tabla` (mismo criterio, sin refactor compartido todavía).
