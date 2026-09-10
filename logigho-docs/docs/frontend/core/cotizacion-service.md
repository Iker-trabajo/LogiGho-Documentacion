---

## Autor: Iker Acevedo

Fecha creacion: 2026-09-09
Estado: produccion

# Servicio: CotizacionService

**Ubicación:** `src/app/core/application/services/cotizacion/cotizacion.service.ts`
**Scope:** `providedIn: 'root'`

---

## ¿Qué hace?

Envuelve la llamada al orquestador de cotizaciones (`ApiLambdaOrquestadorCotizaciones`) y expone los modelos de request/response tipados (`cotizacion.models.ts`). Se extrajo del código inline del modal de creación de pedidos para poder reutilizarlo desde [paso-cotizacion.component](../components/paso-cotizacion.md) sin depender del modal completo.

---

## Métodos

### `cotizar(request: CotizacionRequest): Observable<CotizacionResponse>`

Cotiza contra las 3 transportadoras (o las que vengan en `Proveedores`). El mismo método sirve para la cotización normal (`CotizacionLogigho: true`) y para el desglose crudo (`CotizacionLogigho: false`) — la diferencia la decide quien llama.

| Parámetro | Tipo | Descripción |
| --------- | ---- | ----------- |
| `request` | `CotizacionRequest` | Incluye `Proveedores`, `CotizacionLogigho`, `Common` (con `ConRecaudo`) |

**Retorna:** `CotizacionResponse` con una entrada por transportadora solicitada (`interrapidisimo`, `servientrega`, `envia`), cada una con `data`/`error`.

---

## Endpoints que consume

| Método | Ruta | Descripción |
| ------ | ---- | ----------- |
| `POST` | `/orquestadorCotizacion` | Cotiza en las 3 transportadoras a la vez. Ver [ApiLambdaOrquestadorCotizaciones](../../backend/lambdas-dotnet/aplicacion/Cotizacion/FuncionesCotizar/ApiLambdaOrquestadorCotizaciones/ApiLambdaOrquestadorCotizaciones.md) |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-09-09 | Iker Acevedo | Extracción del código de cotización, antes inline en el modal, a este servicio + `cotizacion.models.ts`. Soporte de `Common.ConRecaudo` en el request. |

---

## Observaciones

- No cachea nada: cada cambio de switch (con/sin recaudo) o de ciudad dispara una llamada nueva.
- `CotizacionLogigho: false` (desglose crudo) usa el mismo método — el gateo por rol vive en el componente que lo llama, no en este servicio.
