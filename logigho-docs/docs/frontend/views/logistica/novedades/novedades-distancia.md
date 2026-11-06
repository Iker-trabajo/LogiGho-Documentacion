---

## Autor: Iker Acevedo

Fecha creacion: 2026-11-06  
Estado: desarrollo (PreProd; producción pendiente)  
Tipo: vista

# Vista: Novedades, distancia a la dirección del pedido

**Selector:** `app-novedades`  
**Ubicación:** `src/app/views/logistica/novedades`  
**Ruta:** `/app/logistica/novedades`  
**Acceso:** Autenticado  
**Historia:** 027 · **Spec:** 012 · **Repo:** `Angular SitioLogiGho`

> Marcas: **observado** (archivo citado), **Iker** (fecha), **hipótesis**. Código leído en `origin/preprod` (PR 620 mergeado) y en la rama
> `feature/novedades-llave-embed` (pendiente de merge). Aquí solo se documenta lo que agregó la historia 027; el resto de la vista (KPIs,
> motivos, evidencias) no se cubre.

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

En la tarjeta del mapa de una guía de Inter, el detalle ofrece dos pestañas (`PestanaMapa`, con roles ARIA `tab`/`tabpanel`):

- **Mensajero:** el mapa del punto donde el mensajero marcó el intento (el iframe `maps.google.com/maps?q=…&output=embed` que ya existía).
- **Comparar con dirección:** para el intento elegido muestra
  - el **mapa embebido con la ruta en carro** desde la dirección del pedido hasta el punto del mensajero (Maps Embed API, modo `directions`),
  - el chip grande con la distancia y el rango del color,
  - «Por calle (en carro)», «En línea recta» y «Precisión de la dirección» (por ejemplo «Exacta (portal)», «Aproximada (por rango de numeración)», «Baja (centro de la zona)», con «coincidencia parcial» si aplica), y
  - el botón **Abrir ruta en Maps** (solo enlaces `https://www.google.com/maps…`).
- Si el intento no tiene coordenada: «Sin dirección: Inter no envió la ubicación de este intento». Con varios intentos, tocar uno cambia el mapa a esa ubicación; en la pestaña «Comparar» también se pueden elegir intentos sin geo.
- La distancia de cada intento se busca por su `HuellaGeo` (y la fecha si hay varias con el mismo punto).

### Llave del mapa embebido

- El mapa de ruta usa una **llave de navegador de Google Maps Embed API**, guardada en `environment.googleMapsEmbedKey` (`environment.ts` y `environment.preprod.ts`).
- La llave es pública por diseño (viaja en el navegador) y por eso va **restringida por dominio (referente) y por API**. **No se escribe su valor en la documentación.**
- Es distinta de la llave de servidor de Distancias (`GOOGLE_MAPS_API_KEY`, cifrada, que nunca llega al navegador).
- Sin llave (vacía) la pestaña muestra solo la distancia, la precisión y el botón «Abrir ruta en Maps» (la URL del mapa se construye solo si hay llave y las coordenadas son válidas: `urlEmbedRuta`).
- **Estado:** la llave está en la rama `feature/novedades-llave-embed`, **pendiente de merge**. Con el PR 620 solo (en `preprod`) la pestaña funciona sin mapa embebido. **Pendiente:** confirmar con Iker que la restricción por dominio incluye el dominio de producción.

---

## Lecturas de Mongo (`metodoGenerico`)

Todo el front sigue por `metodoGenerico` (Iker 2026-11-05). Observado: `helpers/novedades.repository.ts`.

| Dato | Dónde se pide |
| ---- | ------------- |
| `DistanciaUltima` | Proyección del listado de `SeguimientoNovedadesInter` |
| `Distancias` y `DireccionPedido` (solo latitud, longitud, tipo de ubicación, coincidencia parcial y estado) | Proyección del detalle |

No se pide `UltimaRespuesta` ni la dirección del destinatario para la distancia.

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
| 2026-11-06 | Iker Acevedo | Rama `feature/novedades-llave-embed`: llave de navegador para el mapa embebido (pendiente de merge) |

---

## Observaciones

- La vista consume los campos que escribe Distancias; mientras no corra el relleno de producción, las guías de prod mostrarán «Pendiente» (hipótesis).
- Los números del Excel de la vista son los mismos que ve el usuario en pantalla, no los de los reportes de Slack (esos se rigen por [Reglas de los reportes](../../../../backend/lambdas-dotnet/aplicacion/Interrapidisimo/ApiLambdaNovedadesInter/dominio/reglas-reportes.md)).
- **Pendiente:** revisión visual de Iker (vista y Excel) en PreProd.
