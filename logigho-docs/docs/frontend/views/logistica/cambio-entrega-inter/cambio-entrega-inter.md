---

## Autor: Iker Acevedo

Fecha creacion: 2026-08-20
Estado: produccion
Tipo: vista

# Vista: Auditoría Cambio de Entrega a Oficina

**Selector:** `app-cambio-entrega-inter`
**Ubicación:** `src/app/views/logistica/cambio-entrega-inter/`
**Acceso:** Autenticado — scopeado por `tiendas_asignadas` (sessionStorage)

---

## ¿Qué hace?

Le muestra al equipo de logística todas las guías de Interrapidísimo que **nacieron para entrega en dirección** pero que Inter cambió unilateralmente a **"Reclame en oficina"** — y, para las que ya se resolvieron, si terminaron en **entrega** o en **devolución**. Es la cara visual de la auditoría que arma la lambda [`ApiLambdaProcesoAutomaticoInter`](../../../../backend/lambdas-dotnet/aplicacion/interrapidismo/ApiLambaProcesoAutomaticoInter/ApiLambaProcesoAutomaticoInter.md).

El objetivo de negocio: darle a Logística una herramienta para reclamarle a la transportadora con datos concretos — cuántas guías, de qué tiendas, cuánto tiempo llevan en oficina, y cuántas se resolvieron bien o mal.

---

## Ruta

| Ruta | Guard | Parámetros de URL |
| ---- | ----- | ------------------ |
| `/app/logistica/cambio-entrega-inter` | `AuthGuard` | — |

---

## Arquitectura del módulo

Componente raíz "orquestador" — carga los datos, guarda todo el estado en signals (filtros, paginación, modales) y delega el pintado a componentes hijos "tontos" que solo reciben `@Input` y emiten `@Output`.

```
cambio-entrega-inter.component.ts        (orquestador — signals, computed, casos de uso)
├── models/
│   └── cambio-entrega-inter.models.ts   (CambioEntregaInter, filtros, EstadoGestion)
├── repository/
│   └── cambio-entrega-inter.repository.ts (HTTP vía /metodoGenerico, RBAC)
└── components/
    ├── cambio-entrega-filtros/          (tienda/ecosistema, búsqueda, 2 rangos de fecha)
    ├── cambio-entrega-tabla/            (tabla desktop + cards mobile + paginación)
    ├── cambio-entrega-detalle-modal/    (detalle + historial + acciones de contacto)
    ├── cambio-entrega-analisis/         (3 gráficas: distribución/tendencia/tiendas)
    ├── filtro-tienda-ecosistema/        (reutilizado de resumen-inventario)
    └── rango-fecha-calendario/          (reutilizado de pancake-estadistica)
```

Todos los componentes usan `ChangeDetectionStrategy.OnPush` — el módulo carga hasta ~1400 guías completas en memoria (paginación 100% client-side, igual que `gestion-devoluciones`), así que evitar chequeos de cambios innecesarios en toda esa lista es importante para el rendimiento.

---

## Flujo principal

```
ngOnInit()
  -> Promise.all([cargar(), cargarTiendaMap()])   (independientes entre sí)

  cargar()
    -> repository.getCambiosEntrega()              GET /metodoGenerico?coleccion=AuditoriaCambioEntregaInter
    -> ordena por FechaPrimeraDeteccion desc
    -> signal items

  cargarTiendaMap()
    -> repository.getTiendas()                     GET /metodoGenerico?coleccion=Tienda
    -> arma Map<Ecosistema, Tienda[]> filtrado por tiendas_asignadas del usuario

itemsFiltrados (computed)
  -> busqueda + estadoGestion + tienda + 2 rangos de fecha INDEPENDIENTES
     (fecha de detección vs fecha de creación del pedido — ver modelo)

itemsPaginados (computed)
  -> slice de itemsFiltrados según currentPage/pageSize

Acciones del usuario:
  - Clic en KPI                  -> filtrarPorEstadoGestion()
  - Switch de la tabla            -> gestionarRapido()  PUT en memoria + backend, sin recargar todo
  - "Ver detalle"                 -> abrirDetalle()      abre cambio-entrega-detalle-modal
  - "Gráficas"                    -> abrirAnalisis()     abre cambio-entrega-analisis
  - "Exportar"                    -> exportar()          xlsx dinámico, con o sin filtros
  - "?" (tour guiado)             -> tourAbierto = true  ver frontend/components/guided-tour.md
```

---

## KPIs (header)

| KPI | Cálculo |
| --- | ------- |
| Total guías detectadas | `items().length` |
| Pendientes | `EstadoGestion === 'Pendiente'` |
| Gestionadas | `EstadoGestion === 'Gestionada'` |

Clic en cualquiera filtra la tabla por ese estado de gestión.

---

## Filtros

Dos rangos de fecha **independientes** — filtran por campos distintos del modelo, y ambos con su propio calendario:

| Filtro | Campo que filtra | Componente |
| ------ | ------------------ | ---------- |
| Ecosistema / Tienda | `Tienda` (multi-selección) | `filtro-tienda-ecosistema` |
| Búsqueda libre | Guía, destinatario, teléfono, dirección, tienda | input simple |
| Estado de gestión | `EstadoGestion` | `<select>` |
| **Fecha cambio de estado** | `FechaPrimeraDeteccion` — cuándo detectamos el cambio a oficina | `rango-fecha-calendario` |
| **Fecha de creación** | `FechaCreacionPedido` — cuándo se creó el pedido, antes de Inter | `rango-fecha-calendario` |

---

## Exportar a Excel

`xlsx` se importa **dinámicamente** (`await import('xlsx')`) solo cuando el usuario exporta — no engorda el bundle inicial de la pantalla.

Si hay filtros activos, pregunta (SweetAlert2) si exportar lo filtrado o el total; si no hay filtros, exporta directo. Incluye `Estado Final` y `Fecha Resolución` además de todos los campos visibles en la tabla.

Guard de reentrada (`exportando` signal): varios clics seguidos mientras exporta se ignoran, para no disparar la descarga varias veces.

---

## Tour guiado

11 pasos, cubre todo el módulo: KPIs, filtro de tienda/ecosistema, búsqueda, estado de gestión, los 2 filtros de fecha, la tabla (aclarando `Estado` vs `Estado Final`), paginación, y los botones de Gráficas/Exportar/Tour. Ver [Tour Guiado](../../../components/guided-tour.md) para el componente reutilizable.

Nota de implementación: este módulo usa `ChangeDetectionStrategy.OnPush`, y el componente del tour calcula la posición del popover dentro de un `requestAnimationFrame` — un callback que Angular no propaga automáticamente a través de un ancestro `OnPush`. Se corrigió el componente compartido (`ChangeDetectorRef.markForCheck()` después de cada cálculo asíncrono) — el fix beneficia a los 16 módulos que usan el tour, no solo a este.

---

## Responsive

- Tabla: vista de tabla completa en desktop (scroll horizontal, columna "Acciones" fija a la derecha) y cards resumidas en mobile.
- Dropdowns (`filtro-tienda-ecosistema`, popover del calendario, tour guiado): en pantallas ≤480px se anclan fijos abajo tipo *bottom-sheet*, en vez de posicionarse relativos a su botón — evita que se salgan del viewport.
- Todo input/select de este módulo usa `font-size: 16px` en mobile — por debajo de eso, iOS Safari hace zoom automático al enfocar el campo.

---

## Historial de cambios

| Fecha | Autor | Cambio |
| ----- | ----- | ------ |
| 2026-07-31 | Iker Acevedo | Construcción inicial del módulo: modelo, repositorio, tabla, filtros, modal de detalle, RBAC por tienda. |
| 2026-08-15 | Iker Acevedo | Botón "Consultar en plataforma" (deep-link a `/app/tienda/pedidos`), WhatsApp de contacto, bloqueo de scroll del modal. |
| 2026-08-18 | Iker Acevedo | `EstadoFinal`/`FechaResolucion` en el modelo. Modal de gráficas (`cambio-entrega-analisis`) con 3 vistas. Segundo filtro de fecha (creación del pedido). Tour guiado de 11 pasos. Badge de `Estado Final` en tabla y Excel. |
| 2026-08-20 | Iker Acevedo | Filtro de ecosistema dentro del modal de gráficas (con preselección automática si la tabla ya está filtrada por un único ecosistema). Fixes de responsive (bottom-sheet en dropdowns, zoom de iOS, box-sizing del select). Fix del tour guiado para componentes `OnPush` (ver ADR-001). |

---

## Observaciones

- `TotalRecaudo` y `TipoEntrega` llegan como `number` desde el backend — el resto de campos "tipo identificador" (`NumeroGuia`, `Telefono`, `IdTienda`) se mantienen `string` a propósito (ver [ADR-001](adr/ADR-001-desenlace-final.md)).
- La paginación es 100% client-side sobre el dataset completo ya cargado — no hay paginación por servidor en este módulo.
- Ver [ADR-001](adr/ADR-001-desenlace-final.md) para la decisión de diseño detrás de `EstadoFinal` (por qué se persiste en la propia colección de auditoría en vez de cruzar con `PedidosInter` en cada consulta).
