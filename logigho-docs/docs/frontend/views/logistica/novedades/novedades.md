---

## Autor: Yadilton Torres

Fecha creacion: 2026-10-30
Estado: desarrollo
Tipo: vista

# Vista: Novedades

**Selector:** `app-seguimiento-novedades`
**Ubicación:** `src/app/views/logistica/novedades/`
**Acceso:** Autenticado — menú `Modulo` ID `171` (hijo de Logística), roles definidos en la base; datos scopeados por `tiendas_asignadas` (sessionStorage)

---

## ¿Qué hace?

Reúne en una sola pantalla las guías con **novedad abierta** de las tres transportadoras (Interrapidísimo, Envía y Servientrega), para que logística vea qué está fallando, por qué y desde hace cuánto, y pueda contactar al destinatario antes de que el envío se devuelva.

No es un historial: es una **foto del momento**. Una guía aparece mientras su novedad está abierta y sale sola cuando se entrega o se cierra. Por eso el número sube y baja en cada recarga.

Es **solo de lectura**: no escribe en ninguna colección.

---

## Ruta

| Ruta | Guard | Parámetros de URL |
| ---- | ----- | ------------------ |
| `/app/logistica/novedades` | `AuthGuard` + permiso por rol de `Modulo` (layout) | — |

El menú se arma desde la colección `Modulo` (no desde `_nav.ts`): registro `ID: "171"`, `moduloPadre: "3"`, `ruta: "/app/logistica/novedades"`. Para cambiar quién la ve solo se edita `roles` de ese registro; el usuario debe volver a iniciar sesión.

---

## Arquitectura del módulo

Componente raíz que orquesta filtros, KPIs, tabla y panel de detalle. Los datos viven en un store con signals **provisto por la vista** (un store por visita: al salir se descarta con todo lo que tenía). El acceso HTTP está en un único repositorio y la lógica pura en `rules.ts`.

```
novedades.component.ts                  (orquestador — filtros, KPIs, paginación, detalle, Excel)
├── models/
│   └── novedades.models.ts             (NovedadFila, transportadoras, etapa, evidencias)
├── helpers/
│   ├── novedades.repository.ts         (único punto que habla HTTP — metodoGenerico y getObject)
│   ├── novedades.store.ts              (signals: filas, estado de carga, cerradas, detalle, fotos)
│   ├── novedades.mapper.ts             (documentos de Mongo → NovedadFila)
│   ├── novedades.rules.ts              (funciones puras: motivo, semáforo, fechas, guías)
│   └── novedades-excel.helper.ts       (exportación a Excel de lo filtrado)
├── components/
│   └── filtro-desplegable/             (dropdown propio con contador y buscador)
└── styles/
    ├── novedades.scss                  (escritorio)
    └── novedades.mobile.scss           (tablet y celular)
```

Ver: [Modelo](modelos/novedades-models.md) · [Repositorio](servicios/novedades-repository.md) · [Store](servicios/novedades-store.md) · [Mapper](helpers/novedades-mapper.md) · [Reglas](helpers/novedades-rules.md) · [Excel](helpers/novedades-excel.md) · [Filtro desplegable](components/filtro-desplegable.md) · [Panel de detalle](components/detalle-guia.md)

---

## De dónde sale cada transportadora — y por qué son distintas

Cada transportadora **reporta distinto**, así que no se pueden leer igual:

| | Interrapidísimo | Envía | Servientrega |
| - | --------------- | ----- | ------------ |
| Colección | `SeguimientoNovedadesInter` | `PedidosInter` | `PedidosInter` |
| Cómo se encuentra | `EstadoSeguimiento: 'Activo'` (lo mantiene el robot de seguimiento) | `Estado` empieza por `EN NOVEDAD` | `Estado` en `EN PROCESAMIENTO` / `Devolucion` + `Descripcion Asociada` que empieza por `Devolucion:` |
| Motivo | `UltimoMotivo.Motivo` tipificado | Texto tras "Su envío presenta una novedad:" | Texto tras "Devolucion:" |
| Etapa | `novedad` | `novedad` | `devolucion` (ya viene de regreso) |
| Extras | Prioridad, consultas, trazabilidad, mapa por intento, fotos (S3) | Mensaje, PDF, intentos | Motivo de devolución, PDF |

**Servientrega** no escribe la novedad mientras está abierta: su `Estado` en `PedidosInter` es interno (Pagada, Creado, Anulada…) y el motivo solo llega cuando el envío ya se devuelve. Por eso la vista la muestra como **"En devolución"** y no como novedad gestionable. Que aparezca a tiempo depende de que el backend escriba su estado real en `PedidosInter`, como ya pasa con Envía.

---

## Flujo principal

```
ngOnInit()
  store.cargar()
    ├─ repo.listarInter(null)            activas de SeguimientoNovedadesInter
    │    └─ completarIntentosInter()     IntentosEntrega desde PedidosInter (por Numeropreenvio)
    └─ repo.listarPedidos()              Envía + Servientrega en una sola consulta ($or)
    si una fuente falla -> se muestran las demás + aviso de carga parcial
    si las dos fallan   -> pantalla de error con Reintentar

Usuario abre la pestaña Cerradas / Todas
  store.asegurarCerradas()               Inter cerradas de los últimos 7 días (una sola vez)

Usuario abre una guía de Inter
  store.detalle(fila)                    trazabilidad, destinatario, tipo de entrega, PDF, gestion_app
  store.cargarFotos(evidencias)          cada foto desde S3 (wsinter-evidencias-prod) vía getObject

Botón ↻
  store.cargar()                         reemplaza todo y descarta detalle y fotos guardados
```

No hay recarga automática: los datos se cargan al entrar y con ↻. La hora de la última carga se ve en el encabezado y es el "ahora" con que se cuentan los días abiertos.

---

## Pantalla

1. **Encabezado:** título, contador de activas, hora de la última carga, ↻ y **Exportar N guías** (solo lo filtrado).
2. **Selector de transportadora** con logos: Todas / Interrapidísimo / Envía / Servientrega. Define el alcance de KPIs, motivos y tabla. Servientrega muestra "N en devolución".
3. **KPIs clicables:** Novedades activas, Prioridad alta, Más de 3 días, Consultas fallidas. Cada uno filtra la tabla con el mismo número de la tarjeta.
4. **Motivos más frecuentes** en barras clicables (filtran por motivo).
5. **Filtros:** pestañas Activas / Cerradas / Todas, buscador de guías, fecha (Hoy / 3 días / 7 días o rango), Motivo, Estado, Tienda, Prioridad, interruptor "Solo con novedad" y chips de filtros activos.
6. **Tabla** (tarjetas en celular) con barra de semáforo, botón de copiar guía y paginación 5/10/25/50.
7. **Panel de detalle** distinto por transportadora — ver [Panel de detalle](components/detalle-guia.md).

### Buscador de varias guías

Acepta una o varias guías separadas por coma, punto y coma o espacio; cada una puede ser parte del número. Al **pegar una columna de Excel** (una guía por fila) se acomoda sola en una línea con comas, sin vacías ni repetidas. Con dos o más guías, avisa en amarillo cuáles **no están en ninguna novedad cargada**.

### "Solo con novedad"

Encendido por defecto: muestra solo guías con motivo o mensaje. Apagado: suma las que están en seguimiento sin motivo todavía (ej. consulta a Inter pendiente o fallida). Un KPI elegido manda sobre el interruptor.

### Contadores de los desplegables

Cada opción se cuenta **con los demás filtros puestos** (excepto el propio): el número de la opción es exactamente lo que sale al elegirla.

---

## Límites

| Límite | Valor | Por qué |
| ------ | ----- | ------- |
| Tope por consulta | `LIMITE_CONSULTA = 5000` | Freno de seguridad; si se alcanza, aviso de carga parcial |
| Cerradas de Inter | `DIAS_CERRADAS = 7` días | Se acumulan por miles; solo se piden al abrir su pestaña |
| Tiendas | `tiendas_asignadas` | Si el usuario no tiene "Todas", cada consulta filtra por sus tiendas |
| Detalle de Inter | Bajo demanda | Trazabilidad, destinatario y fotos son pesados y traen datos personales |

---

## Seguridad y datos personales

- Todas las consultas son **de lectura**: `metodoGenerico` con pipeline (la Lambda bloquea `$out`, `$merge`, `$lookup`, `$unionWith`) y `getObject` para las fotos.
- El listado usa una proyección liviana: **sin** `UltimaRespuesta` completa. Destinatario y trazabilidad de Inter se piden solo al abrir el detalle y se descartan en cada recarga.
- **Nada se guarda en el navegador** (ni sessionStorage, ni localStorage, ni IndexedDB). La memoria se libera al salir de la vista.
- El **Excel no incluye** datos del destinatario (nombre, teléfono, dirección) ni fotos.
- El acceso lo dan los roles del registro `171` en `Modulo`; las tiendas visibles las da `tiendas_asignadas`.

---

## Responsive

`styles/novedades.mobile.scss`, con las mismas técnicas que `relacion-despacho.mobile.scss`:

- **≤1180px:** KPIs en una fila y motivos debajo.
- **≤860px:** encabezado y filtros en columna, KPIs 2×2, transportadoras deslizables.
- **≤768px (celular):** vista a todo el ancho, sin cajas dentro de cajas; resumen con solo las 4 tarjetas y **"Ver más"** para los motivos; botón **Filtros** que abre una hoja a pantalla completa (fechas con el calendario nativo del teléfono, pie fijo con "Ver N guías"); desplegables como hoja inferior; tarjetas en vez de tabla; detalle a pantalla completa.

El esqueleto de carga mide lo mismo que el contenido real para que la tabla no salte al llegar los datos, y las filas entran con un desvanecido corto sin desplazamiento.

---

## Pruebas

52 pruebas (`ng test --include="src/app/views/logistica/novedades/**/*.spec.ts"`): reglas, mapper, store, Excel y componente (filtros, KPIs, contadores, rangos de fecha, página 1 al filtrar, varias guías, pegado de Excel, mapa por intento, fotos de S3, Esc). Las pruebas usan filas mínimas definidas dentro de cada `.spec.ts`; la vista no tiene datos de ejemplo.

---

## Historial de cambios

| Fecha | Autor | Cambio |
| ----- | ----- | ------ |
| 2026-10-28 | Yadilton Torres | Construcción inicial: vista visual (encabezado, transportadoras, KPIs, motivos, filtros, tabla, panel de detalle) y conexión a datos reales de solo lectura (Inter, Envía), Excel de lo filtrado. |
| 2026-10-30 | Yadilton Torres | Versión para celular (hoja de filtros, "Ver más", desplegables como hoja inferior); Servientrega en devolución; detalle distinto por transportadora; intentos de entrega con mapa; fotos de la novedad desde S3; búsqueda de varias guías y pegado desde Excel; corrección de 8 fallas (contadores de filtros, rangos de fecha, página al filtrar, salto de la tabla al cargar, entre otras); quitados los datos de ejemplo; menú `Modulo` ID 171. |

---

## Observaciones

- **Servientrega:** sus 185 guías "en devolución" incluyen algunas de meses atrás que quedaron en `EN PROCESAMIENTO`/`Devolucion` en la base; se dejaron visibles a propósito. Es un tema de datos del backend.
- **Envía** a veces trae `EN NOVEDAD` con un mensaje de "Su envío está viajando a la ciudad de destino", que no parece una novedad real.
- Los mismos pedidos de Servientrega que no actualizan su `Estado` explican por qué aparecen guías ya despachadas en **Sin Despacho**: siguen en `Creado` en `PedidosInter`.
- Pendiente del backend: índices en `SeguimientoNovedadesInter` para `Tienda` y `EstadoSeguimiento + FechaActualizacion`.
