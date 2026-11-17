---

## Autor: Iker Acevedo

Fecha creacion: 2026-11-06  
Estado: desarrollo (PR 637 en `preprod` desde 2026-11-08; no hay evidencia de que esté ya en `main`)  
Actualizado: 2026-11-09 (PR 637)  
Tipo: vista

# Vista: Novedades, distancia a la dirección del pedido

**Selector:** `app-novedades`  
**Ubicación:** `src/app/views/logistica/novedades`  
**Ruta:** `/app/logistica/novedades`  
**Acceso:** Autenticado  
**Historia:** 027 · **Spec:** 012 · **Repo:** `Angular SitioLogiGho`

> Marcas: **observado** (archivo citado), **Iker** (fecha), **hipótesis**. Código leído en `origin/preprod` (PR 620, 628 y 637 mergeados; sin diferencias con la rama de trabajo
> `feature/novedades-mejoras-visuales` en esta carpeta el 2026-11-09). Aquí solo se documenta lo que agregó la historia 027; el resto de la vista (KPIs,
> motivos, evidencias) no se cubre. Si el código contradice una versión anterior de esta página, manda el código (ver [Observaciones](#observaciones)).

---

## ¿Qué hace?

Para las guías de Interrapidísimo muestra a cuántos metros por calle (en carro) de la dirección del pedido marcó el mensajero el intento, con un
semáforo, y permite comparar el punto del mensajero con la dirección en un mapa. Los datos los calcula la función
[Distancias](../../../../backend/lambdas-dotnet/aplicacion/Interrapidisimo/ApiLambdaNovedadesInter/ApiLambdaNovedadesInterDistancias.md); la vista solo los lee.
Envía y Servientrega no traen coordenada: la columna no aplica y queda vacía.

---

## Columna de distancia y chip

- Columna **Distancia** en la tabla (escritorio) y chip equivalente en las tarjetas (móvil). Observado: `novedades.component.html`.
- El chip lleva color según los metros por calle (`novedades.rules.ts`, `METROS_VERDE_MAX = 250`, `METROS_AMARILLO_MAX = 500`):

| Metros | Chip |
| ------ | ---- |
| ≤ 250 | Verde |
| 251 a 500 | Amarillo |
| > 500 | Rojo |

- Se usa el color que calculó el backend si es válido; si no, sale de los metros (`colorDeDistancia`). El formato es es-CO con punto de miles (`1.240 m`).
- Cada chip tiene `aria-label` con la distancia y el color.

### Etiquetas sin metros

| Etiqueta | Cuándo |
| -------- | ------ |
| **Sin dirección** | El intento no trae coordenada (`SinGeo`). También se muestra si el último motivo no tiene `geoUrl` y aún no hay distancia |
| **Dirección por validar** | `DireccionNoEncontrada` o `BajaConfianza` (incluye el rojo no confirmado) |
| Error al calcular | `Error`, o `Calculada` sin metros |
| Pendiente | Documento sin el campo (aún no procesado por la función) |

Hay además un botón con el contador de guías «sin dirección» que activa ese filtro.

---

## Filtro de distancia (5 opciones)

Desplegable «Distancia» (`app-filtro-desplegable`), con contador por opción y chip de filtro activo:

| Opción | Valor interno |
| ------ | ------------- |
| ≤ 250 m | `verde` |
| 251–500 m | `amarillo` |
| > 500 m | `rojo` |
| Sin dirección | `sin-direccion` |
| Dirección por validar | `por-validar` |

«Todas» (valor vacío) deja pasar todo, incluidas Envía y Servientrega. Observado: `OPCIONES_DISTANCIA` y `cumpleFiltroDistancia` en `novedades.rules.ts`.

---

## Detalle: pestañas «Mensajero» y «Comparar con dirección»

En la tarjeta del mapa de una guía de Inter, el detalle ofrece dos pestañas (`PestanaMapa`, con roles ARIA `tab`/`tabpanel`). Desde el PR 637 el mapa es **propio**
(`components/mapa-ruta/mapa-ruta.component.ts`, Maps JavaScript API; observado):

- **Mensajero:** un marcador con el logo de Inter en el punto donde marcó el intento, sin ruta.
- **Comparar con dirección:** pin rojo en la dirección del pedido, marcador de Inter en el punto del mensajero y una **línea recta punteada de referencia**. **No se pide ninguna ruta a Google desde el navegador**
  (decisión de Iker 2026-11-08: la llave del navegador no tenía Routes y cada apertura era un cobro). La distancia por calle ya la guarda el backend y el recorrido real se ve con el botón **Abrir ruta en Maps**
  (solo enlaces `https://www.google.com/maps…`; viene de `DistanciaUltima.UrlRuta`). Muestra además el chip grande con la distancia, «Por calle», «En línea recta» y la precisión de la dirección.
- Con `conRuta = false` (dirección por validar) se marcan los dos puntos pero no se dibuja la línea.
- Si no hay llave, el script no carga o Google rechaza la llave (`gm_authFailure`), la vista cae al iframe `maps.google.com/maps?q=…&output=embed` de siempre.
- Si el intento no tiene coordenada: «Sin dirección: Inter no envió la ubicación de este intento». Con varios intentos, tocar uno cambia el mapa a esa ubicación.
- La distancia de cada intento se busca por su `HuellaGeo` (y la fecha si hay varias con el mismo punto).
- **Pendiente (M-276):** dibujar el recorrido real por calle exigiría guardar el trazado (polyline) en el backend y recalcular ~8 000 rutas con costo extra de Google; no se decidió.

### Dirección del repartidor (v8)

La dirección en texto del punto del mensajero sale de `SeguimientoNovedadesInter.DireccionMensajero.Texto` cuando su `HuellaGeo` **coincide con la del último punto** del intento (costo USD 0). El mapper
(`direccionMensajeroDesdeDoc`) solo la acepta con `Estado = Calculada`, texto no vacío y una huella válida; el texto se muestra como texto, nunca como HTML. Para intentos **anteriores** al último se mantiene la geocodificación
inversa del navegador (`helpers/direccion-inversa.ts`: una llamada por coordenada, guardada en memoria; sin llave, no se muestra). Decisión y alternativas: [ADR-006](../../../../backend/lambdas-dotnet/aplicacion/Interrapidisimo/ApiLambdaNovedadesInter/adr/ADR-006-direccion-del-mensajero-guardada-por-geocodificacion-inversa.md).

### Llave de navegador

- Vive en `environment.googleMapsEmbedKey` (`environment.ts` y `environment.preprod.ts`). Aunque se llama «Embed», hoy sirve para cargar la **Maps JavaScript API** y la geocodificación del navegador.
- Es pública por diseño (viaja en el navegador) y va **restringida por dominio (referente) y por API**. **No se escribe su valor en la documentación.** Es distinta de la llave de servidor de Distancias (`GOOGLE_MAPS_API_KEY`, cifrada).
- **Pendiente (se arrastra del 2026-11-06):** confirmar con Iker que la restricción por dominio incluye el dominio de producción.
- M-269: `google.maps.Marker` está deprecado; migrar a `AdvancedMarkerElement` (requiere Map ID).

---

## Estados Logigho e Interrapidísimo en el panel (v8)

El panel de detalle de una guía de Inter muestra **dos** estados distintos (observado: `novedades.component.html`):

| Etiqueta | Campo | Qué es |
| -------- | ----- | ------ |
| **Estado Logigho** | `PedidosInter.Estado` (se lee con el detalle) | El estado del pedido en Logigho (Intento de entrega, Devolución Completada, Pagada, …). `—` si no hay pedido; `…` mientras carga el detalle |
| **Estado Interrapidísimo** | `UltimaRespuesta.estado_actual`, con respaldo en `UltimoMotivo.EstadoInter` | El estado actual de la guía en Inter (el mismo que usan los Excel de Slack) |

Para Envía y Servientrega solo se muestra «Estado».

---

## Cerradas de Inter: periodo y cursor por `_id` (v8, Iker 2026-11-08)

- **Periodo elegible:** 7 días, 30 días, 90 días o «Todo el historial» (`PERIODOS_CERRADAS`, `PeriodoCerradas = 7 | 30 | 90 | 0`; 0 = todo). Lo ya traído no se vuelve a pedir: un periodo mayor cubre a los menores (`cubre()`).
- **Cursor por clave (`_id`) en lugar de `$skip`:** cada tanda pide `_id > último recibido`, ordenada por `_id`; no se vuelve a recorrer ni ordenar lo anterior. **Tandas de 3 000** (`TANDA_CERRADAS`;
  medido contra prod 2026-11-08: 1 000 → ~0,5 s, 2 000 → ~0,8 s, 3 000 → ~1,1 s, 5 000 → ~1,7 s por tanda). Tope de seguridad de 100 tandas.
- **Tres cursores paralelos** (`MAX_CURSORES = 3`) para «Todo el historial» sin otro filtro y con al menos 6 000 filas: un conteo y un corte por `_id` reparten el recorrido en tramos iguales (**~4,9 s para 17 124 cerradas**, contra ~6,9 s con un solo cursor).
  Si algo falla al calcular los cortes, se usa un cursor.
- **Lista liviana:** sin `UltimaRespuesta` ni `Historial` (los intentos se piden con el detalle) y solo los subcampos que usa la vista; el cruce de intentos con `PedidosInter` es perezoso, en tandas de 400.
- Sin `$facet`, `$lookup` ni `$unionWith` (DocumentDB). Todo por `metodoGenerico`.
- Atajo **«Traer todo el historial con esta distancia»** (`traerCerradasDeDistancia`): trae del servidor las cerradas de los rangos de distancia elegidos. M-277: su regla de «Sin dirección» (`DistanciaUltima.Estado = 'SinGeo'`) no coincide del todo con la regla local.

### Cifras que dependen del usuario (M-279)

Las consultas filtran por las **tiendas asignadas** al usuario (`sessionStorage` `tiendas_asignadas`; si trae «Todas», no se filtra; observado: `novedades.repository.ts`). Por eso las cifras de la vista **no coinciden**
con las de los reportes de Slack, que traen todas las tiendas (el 2026-11-08 la cuenta de prueba tenía 205 tiendas). La nota del filtro de distancia aclara que los conteos son de las guías cargadas (activas y cerradas del periodo elegido).

---

## Filtro de distancia de selección múltiple (v8)

El desplegable «Distancia» ahora es de **selección múltiple** (`[multiple]="true"`, `elegirDistancias`): se pueden combinar, por ejemplo, «> 500 m» y «Dirección por validar». Los conteos por opción se calculan sobre lo cargado.
Las opciones y sus valores internos no cambian (tabla de arriba).

---

## Lecturas de Mongo (`metodoGenerico`)

Todo el front sigue por `metodoGenerico` (Iker 2026-11-05). Observado: `helpers/novedades.repository.ts`.

| Dato | Dónde se pide |
| ---- | ------------- |
| `DistanciaUltima` (incluye `Modo` y `HuellaGeo`) | Proyección del listado de `SeguimientoNovedadesInter` |
| `UltimaRespuesta.estado_actual` (como `EstadoActual`) | Listado (solo ese subcampo) |
| `Distancias`, `DireccionPedido` (latitud, longitud, tipo de ubicación, coincidencia parcial y estado) y `DireccionMensajero` (`Estado`, `Texto`, `HuellaGeo`) | Proyección del detalle |
| `PedidosInter.Estado`, `Direccion`, `Ciudad`, `Departamento`, `UrlGuia` | Detalle (Estado Logigho) |

No se pide `UltimaRespuesta` completa ni los datos del destinatario para la distancia.

---

## Excel de la vista

La descarga de la vista (`novedades-excel.helper.ts`) suma cuatro columnas al final (solo Inter; vacías en Envía y Servientrega):

| Columna | Contenido |
| ------- | --------- |
| `Distancia (m)` | Metros redondeados, si está calculada |
| `Color` | Verde, Amarillo o Rojo |
| `Estado distancia` | `Calculada` o la etiqueta («Sin dirección», «Dirección por validar», …) |
| `Ruta en Maps` | Solo enlaces `https://www.google.com/maps…`; si no, vacío |

El archivo no lleva datos del destinatario (nombre, teléfono, dirección).

---

## Reglas (`helpers/novedades.rules.ts`)

| Función | Qué calcula |
| ------- | ----------- |
| `colorPorMetros` / `colorDeDistancia` | Color por metros o el del backend |
| `categoriaDistancia` | `no-aplica`, `pendiente`, `calculada`, `sin-direccion`, `por-validar`, `error` |
| `etiquetaDistancia` | Texto de la celda y del Excel |
| `cumpleFiltroDistancia` | ¿La guía pasa el filtro? |
| `distanciaDeIntento` | Distancia que corresponde a un intento (por `HuellaGeo` y fecha) |
| `urlRutaSegura` | Valida que el enlace sea de `https://www.google.com/maps` |
| `urlEmbedRuta` | URL del Embed API, o `null` sin llave o con coordenadas inválidas |
| `precisionDireccion` | Texto de la precisión de la geocodificación |

Pruebas: `novedades.rules.spec.ts`, `novedades.store.spec.ts`, `novedades.mapper.spec.ts`, `novedades-excel.helper.spec.ts`, `novedades.component.spec.ts` (81 specs, build de PreProd; spec 012 2026-11-05).

---

## Seguridad

- Los enlaces solo se aceptan si empiezan por `https://www.google.com/maps` (host `www.google.com`); cualquier otro se descarta.
- Las URLs del iframe se sanean una sola vez por URL (caché) y solo se arman con coordenadas válidas.
- El Excel no incluye datos personales.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-11-05 | Iker Acevedo | PR 620 (`preprod`): columna, filtro, etiquetas, pestañas y Excel |
| 2026-11-06 | Iker Acevedo | Rama `feature/novedades-llave-embed`: llave de navegador para el mapa embebido |
| 2026-11-09 | Iker Acevedo (redactado con IA) | PR 637 (2026-11-08): dirección del repartidor desde el backend, mapa sin Routes API, periodo de cerradas y cursor por `_id`, filtro de distancia múltiple, panel con Estado Logigho e Interrapidísimo |

---

## Observaciones

- La vista consume los campos que escribe Distancias; mientras no corra el relleno de producción, las guías de prod mostrarán «Pendiente» (hipótesis).
- Los números del Excel de la vista son los mismos que ve el usuario en pantalla, no los de los reportes de Slack (esos se rigen por [Reglas de los reportes](../../../../backend/lambdas-dotnet/aplicacion/Interrapidisimo/ApiLambdaNovedadesInter/dominio/reglas-reportes.md)).
- **Pendiente:** revisión visual de Iker (vista y Excel) en PreProd.
- **Discrepancia corregida (2026-11-09):** esta página decía que la pestaña «Comparar» usaba el **Maps Embed API en modo `directions`** (iframe con ruta). El código actual (PR 637) usa el mapa propio con línea recta y no pide rutas.
  `urlEmbedRuta` sigue en `novedades.rules.ts` y el componente aún lo calcula (`novedades.component.ts`, línea 963); es el respaldo del iframe cuando falla el mapa propio (hipótesis: no se probó cada rama).
- **Comentario obsoleto en el código:** `listarCerradasInter` dice «Orden estable por `FechaCierre` y `Guia` para que `$skip` no repita»; el cuerpo hoy usa cursor por `_id` (mejora de limpieza, sin impacto funcional).
- M-278: «Todo el historial» usa 3 cursores en paralelo; no se probó bajo carga ni se midió el tamaño comprimido. Si molesta a DocumentDB, volver a 1 cursor (~6,9 s).
- M-280: los PR del front a `main` repetirán el conflicto «both added» porque los PR 625 y 628 entraron por squash a `preprod`; el PR 637 se resolvió con `merge -s ours` tras comprobar que `preprod` era idéntico al commit 94a5db33 de la rama (opción B de Iker 2026-11-08).
- Los PR de esta vista: 620, 625, 628 y 637 (Angular `preprod`).
