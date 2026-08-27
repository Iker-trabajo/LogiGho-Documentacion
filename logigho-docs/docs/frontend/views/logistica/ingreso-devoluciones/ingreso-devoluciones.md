---

## Autor: Iker Acevedo

Fecha creacion: 2026-08-26
Estado: produccion
Tipo: vista

# Vista: Ingreso Devoluciones

**Selector:** `app-ingreso-devoluciones`
**Ubicación:** `src/app/views/logistica/ingreso-devoluciones/`
**Acceso:** Autenticado — scopeado por `tiendas_asignadas` (sessionStorage)

---

## ¿Qué hace?

Reemplaza al módulo legacy de devoluciones masivas (pistoleo manual guía por guía contra `PedidosInter` directo desde el navegador, sin lotes ni control). Este módulo consume el backend nuevo [`ApiLambdaDevolucionesMasivo`](../../../../backend/lambdas-dotnet/aplicacion/Logigho/ApiLambdaDevolucionesMasivo/ApiLambdaDevolucionesMasivo.md): sube un archivo o pistolea guías, sigue el avance en vivo mientras el servidor procesa por lotes, y revisa el historial completo de todo lo cargado — cuáles guías se aceptaron y cuáles fallaron, con el motivo exacto y qué hacer con la caja física.

El objetivo de negocio es el mismo que el legacy (marcar pedidos como `"Devolucion Completada"`), pero sin los picos de 99% de CPU en DocumentDB que causaba procesar todo desde el navegador sin lotes ni límite de conexiones.

---

## Ruta

| Ruta | Guard | Parámetros de URL |
| ---- | ----- | ------------------ |
| `/app/logistica/ingreso-devoluciones` | `AuthGuard` | — |

---

## Arquitectura del módulo

Componente raíz "orquestador" — arma el polling, decide cuándo mostrar cada bloque, y delega el pintado a componentes hijos. El estado compartido vive en un store con signals, separado de la lógica pura (funciones sin dependencias, 100% testeables) y del acceso HTTP (un único repositorio).

```
ingreso-devoluciones.component.ts        (orquestador — polling, persistencia de sesion, tour)
├── models/
│   └── ingreso-devoluciones.models.ts   (contratos API + documentos Mongo + catalogo de motivos)
├── helpers/
│   ├── ingreso-devoluciones.repository.ts  (unico punto que habla HTTP)
│   ├── ingreso-devoluciones.store.ts       (signals: lote activo, historial, detalle)
│   ├── ingreso-devoluciones.rules.ts       (funciones puras: fechas, paginacion, motivos)
│   └── escaneo-buffer.ts                   (agrupa escaneos antes de enviarlos)
└── components/
    ├── carga-archivo/       (banner + drag&drop + plantilla)
    ├── panel-escaneo/       (pistoleo con lector fisico o camara)
    ├── progreso-lote/       (progreso en vivo del archivo grande)
    ├── historial-lotes/     (tabla, buscador global, cada fila despliega su detalle)
    ├── detalle-lote/        (aceptadas/rechazadas de un lote, pestañas + filtros + export)
    └── filtro-multiple/     (dropdown de seleccion multiple, reutilizable)
```

**Por qué `repository`, `store` y `rules` están separados** (Single Responsibility a nivel de capa): el repositorio solo sabe hablar HTTP, el store solo guarda estado, las reglas son funciones puras sin ninguna dependencia. Es la misma separación que "el que va al mercado, la despensa y las recetas" — se puede testear una receta sin mockear el mercado. `ingreso-devoluciones.rules.spec.ts` tiene decenas de tests sobre funciones puras sin ningún mock de Angular ni de red.

---

## Dos fuentes de datos con formas distintas — y por qué

El módulo consume **dos APIs distintas a propósito**, documentado en la cabecera de `ingreso-devoluciones.models.ts`:

| | La API de lambdas (`POST /iniciar`, `GET /estado`) | `metodoGenerico` (CRUD genérico) |
| - | --------------------------------------------------- | --------------------------------- |
| Uso | Polling rápido mientras el lote está activo | Historial cerrado, detalle de aceptadas |
| Nulos | Se **omiten** del JSON (`DefaultIgnoreCondition.WhenWritingNull`) — por eso los tipos van con `?`, no `| null` | Llegan explícitos como `null` |
| Fechas | ISO plano | Envueltas como `{ $date: "..." }` |
| Por qué existe | `GET /estado` solo devuelve jobs **abiertos** — nunca expone el historial cerrado | Es la única forma de leer `JobsDevolucion`/`InventarioDevolucion` completos |

`parseFecha()` en `rules.ts` normaliza las dos formas de fecha en un solo punto, para que ningún componente tenga que saber de dónde vino el dato.

---

## Flujo principal

```
ngOnInit()
  reengancharSiHayLoteAbierto()
    repo.obtenerAbiertos()                     jobs Pendiente/EnProceso del usuario
    si hay uno -> store.setLoteActivo() + iniciarPolling(jobId)
                  recupera guiasEnviadas de sessionStorage si el jobId coincide
  cargarHistorial(usuarioEmail)                metodoGenerico?coleccion=JobsDevolucion

Operario sube un archivo (carga-archivo) o pistolea (panel-escaneo)
  -> iniciarLote(guias)
       repo.iniciarLote(guias)                  POST /iniciar
       store.setLoteActivo(estado minimo)        mientras llega el primer poll real
       persistirLotePersistido(jobId, guias)     sessionStorage — sobrevive a recargar el componente
       iniciarPolling(jobId)

iniciarPolling(jobId)                            while loop, NO interval()/switchMap()
  repo.obtenerEstado(jobId)  cada INTERVALO_POLLING_MS (2.5s), esperando la respuesta antes de agendar la siguiente
  store.setLoteActivo(resp)
  si Completado/Fallido -> cargarResultadoFinal(jobId)

cargarResultadoFinal(jobId)
  repo.obtenerEstado(jobId, true)                UNA vez, trae Detalle (rechazadas)
  repo.obtenerInventarioDeLote(guiasEnviadas)     aceptadas de ESTE lote
  cargarHistorial(usuarioEmail)                  el lote recien terminado aparece de primero
  store.setLoteActivo(null)                      vuelve a la pantalla de carga
  limpiarLotePersistido()
```

---

## Por qué NO se usa `interval()` + `switchMap()` de RxJS para el polling

Decisión deliberada, documentada en el código: con un `interval(2500)` fijo, si una respuesta del backend tarda más de 2.5 segundos (pasa con cold starts de Lambda), `switchMap` **cancela esa petición a mitad de vuelo** y la pierde en silencio — el front queda un ciclo entero sin saber qué pasó. Un loop `while` con `async/await` que espera la respuesta antes de agendar el siguiente `setTimeout` nunca superpone peticiones y nunca pierde una. Es más código que un `.pipe()`, pero es correcto.

---

## Persistencia de sesión: sobrevivir a que el componente se destruya

`EstadoLoteResponse` solo trae **contadores**, nunca la lista de guías enviadas — necesaria para filtrar `InventarioDevolucion` al terminar el lote. Si el operario navega a otra pantalla y vuelve (el componente shell se destruye y recrea, aunque la pestaña siga abierta), esa lista se perdería.

Solución: `sessionStorage` guarda `{ jobId, guias }` justo al iniciar el lote. Al reenganchar (`reengancharSiHayLoteAbierto`), si el `jobId` persistido coincide con el del job abierto que trae `obtenerAbiertos()`, se recupera la lista. Se limpia cuando el lote termina.

**Límite aceptado:** si se cierra la pestaña completa (no solo se navega dentro del sitio), `sessionStorage` no sobrevive — el reenganche seguiría trayendo el estado real del job desde el backend, pero sin la lista de guías, así que el detalle de "aceptadas" de ese lote específico no se podría filtrar al terminar (el historial sigue mostrándolo bien, porque desde [ADR-003](../../../../backend/lambdas-dotnet/aplicacion/Logigho/ApiLambdaDevolucionesMasivo/adr/ADR-003-jobid-en-inventariodevolucion.md) esa consulta ya no depende de la lista en memoria, sino del `JobId` persistido en cada fila).

---

## El pistoleo es un flujo independiente del archivo grande

`panel-escaneo` **nunca** toca `store.loteActivo()` — tiene su propio buffer (`EscaneoBuffer`) que agrupa escaneos y dispara sus propios mini-jobs, con su propio mini-polling. Esto es intencional: un operario puede tener un archivo de 2.000 guías procesándose en el fondo y seguir pistoleando cajas sueltas al mismo tiempo, sin que se pisen entre sí. Ver [`panel-escaneo`](components/panel-escaneo.md).

---

## Seguridad

- El JWT viaja en el header `Token` (no `Authorization`), leído por `ConsumoGenericoService` — ver [`LectorTokenCognito`](../../../../backend/lambdas-dotnet/aplicacion/Logigho/ApiLambdaDevolucionesMasivo/ApiLambdaDevolucionesMasivo.md#seguridad) del lado del backend.
- El filtro por tienda es del lado del **backend**: `ReglaPermisoTienda` rechaza cualquier guía de una tienda que el usuario no tenga asignada, antes de que el front la vea. El front además filtra `InventarioDevolucion` por `NombreTienda` (sessionStorage `tiendas_asignadas`, sin filtro si el usuario tiene `"Todas"`) al consultar el detalle de aceptadas — doble capa, la que de verdad protege es la del backend.
- Un job solo lo puede consultar su dueño — el backend responde `404` (no `401`) para un `jobId` ajeno, ver [`ConsultarJobUseCase`](../../../../backend/lambdas-dotnet/aplicacion/Logigho/ApiLambdaDevolucionesMasivo/casos-uso/consultar-job-usecase.md).

---

## Tour guiado

6 pasos, cubre el flujo completo: subir archivo, pistolear, revisar el historial, abrir un lote, entender los motivos de rechazo, y el recordatorio de que se puede cerrar la pestaña mientras un archivo grande procesa. Ver [Tour Guiado](../../../components/tour-guiado.md) para el componente reutilizable.

**Detalle de implementación importante:** `TourGuiadoComponent` resuelve qué pasos existen **una sola vez**, al abrir el tour (`steps.filter(s => !!document.getElementById(s.elementId))`) — un paso que apunte a un elemento que todavía no está en el DOM se descarta en silencio, no se reintenta después. Dos consecuencias en este módulo:

1. El paso que explica el detalle de rechazadas apunta a `id-tour-rechazos`, que solo existe con un lote desplegado en el historial. `abrirTour()` llama primero a `historial.expandirPrimero()` y espera un `setTimeout` (un tick) antes de abrir el tour, para que Angular alcance a pintar esa fila.
2. Cada paso usa `onBeforeStep` para dejar la pantalla en el estado correcto (abrir/cerrar el detalle, abrir/cerrar el pistoleo) — incluyendo el paso 5, que vuelve a abrir el detalle por si el operario llega ahí retrocediendo desde el paso 6 (que lo cierra).

---

## Responsive

- El módulo no tiene una vista mobile dedicada — se apoya en el layout general de CoreUI (sidebar colapsable). Las tablas (`historial-lotes`, `detalle-lote`) usan scroll horizontal contenido, no cards mobile.
- El panel de escaneo sí distingue mobile: en celular muestra el botón de cámara (ZXing) en vez del input de teclado, porque en ese contexto no hay lector físico de código de barras.

---

## Historial de cambios

| Fecha | Autor | Cambio |
| ----- | ----- | ------ |
| 2026-08-25 | Iker Acevedo | Construcción inicial: modelos, repositorio, store, reglas puras, componentes de carga/progreso/detalle/historial, integración con el backend nuevo. |
| 2026-08-26 | Iker Acevedo | Rediseño visual (banner, tabs, tablas centradas), fusión de "Subir archivo" e "Historial" en una sola vista, expansión de fila en el historial en vez de navegación, filtro múltiple de transportadoras reutilizable, descarga de plantilla real desde S3, tour guiado de 6 pasos, correlación exacta de aceptadas por `JobId`. |

---

## Observaciones

- El historial y la carga de archivo conviven en la misma pantalla a propósito — el diseño original tenía pestañas separadas para "Subir archivo" / "Historial", pero se fusionaron para que el operario vea de inmediato lo que ya procesó sin cambiar de pestaña, tal como en el mockup de referencia del diseño.
- `guiaOriginalDe()` en `rules.ts` prefiere la guía que reportó la transportadora, y si no existe, cae a `Numeropreenvio` (que sale de `PedidosInter` directamente) — esto permite mostrar la guía original incluso en motivos como `PedidoAnulado`/`SinPermisoTienda`, donde el pedido sí se encontró y nunca hizo falta consultarle nada a la transportadora.
- Ver [detalle-lote](components/detalle-lote.md) para la nota sobre por qué los datos derivados se calculan en campos, nunca en getters del template — un bug real de esta implementación que rompía la selección de texto con el mouse.
