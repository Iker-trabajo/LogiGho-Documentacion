---

## Autor: Iker Acevedo

Fecha creacion: 2026-09-09
Estado: produccion
Tipo: componente

# Componente: PasoCotizacionComponent

**Selector:** `app-paso-cotizacion`
**Ubicación:** `src/app/components/modal-creacion-pedidos/components/paso-cotizacion/`
**Acceso:** Autenticado | paso 3 del `ModalCreacionPedidosComponent`

---

## ¿Qué hace?

Es el paso 3 del [modal de creación de pedidos](modal-creacion-pedidos.component.md): cotiza el envío contra Interrapidísimo, Servientrega y Envía a la vez, y muestra las 3 tarifas como tarjetas para que el usuario elija. Antes vivía todo inline dentro del modal; se extrajo a componente propio para poder tocarlo sin arriesgar el resto del flujo.

---

## Con/sin recaudo

Switch "Con recaudo / Sin recaudo" que aplica a las **3** transportadoras a la vez (`Common.ConRecaudo` en el request al orquestador). Antes solo existía visualmente para Envía — el campo ya era global en el payload, pero el backend lo ignoraba para Inter/Servientrega hasta la Ronda 2 de `feature/integracion-cotizador-envia`. Ver [ApiLambdaOrquestadorCotizaciones](../../backend/lambdas-dotnet/aplicacion/Cotizacion/FuncionesCotizar/ApiLambdaOrquestadorCotizaciones/ApiLambdaOrquestadorCotizaciones.md).

---

## UI

- **Tarjeta por transportadora** con badge **"Tarifa más económica"** — se calcula en el front comparando los 3 totales devueltos, no viene del backend.
- El badge de trayecto (`cubrimiento`) que se mostraba para Envía **se quitó** por decisión del usuario. El backend lo sigue devolviendo, pero no se renderiza.
- Nota fija: "el valor declarado es la base con la que se cotiza el flete".

### Botón "Ver desglose por transportadora"

Visible **solo para roles CEO/Desarrollador**, gateado por `sessionStorage.getItem('roles_asignados')` — mismo patrón que usan `exportacion-pedidos.component.ts` y `floating-bar.component.ts` en el resto del repo.

Al abrirlo dispara una **segunda cotización** con `CotizacionLogigho: false`, para mostrar el desglose crudo que devuelve cada transportadora (flete base, seguro, etc.) sin la fórmula Logigho aplicada encima.

> ⚠️ **Esto es ocultamiento de UI, no control de acceso.** El backend no valida el rol de quien llama al endpoint del orquestador — cualquiera con el token puede pedir `CotizacionLogigho: false` directamente. Si en algún momento el desglose crudo se considera información sensible de verdad, hay que validarlo en el backend, no confiar en este gate del front.

### Labels del desglose

Ajustados a pedido de negocio — antes cada transportadora tenía su propio nombre para el mismo concepto, ahora todos dicen **"Seguro"**:

| Transportadora | Label anterior | Label actual |
| --------------- | --------------- | ------------ |
| Servientrega | "Sobreflete" | "Seguro" |
| Envía | "Servicio" (antes también "Costo M"/"Otros") | "Seguro" |

---

## Servicios y endpoints

| Servicio | Método | Endpoint | Cuándo |
|---|---|---|---|
| `CotizacionService` | `cotizar()` | `POST /orquestadorCotizacion` (`CotizacionLogigho: true`) | Cotización normal, al entrar al paso o cambiar switch/ciudades |
| `CotizacionService` | `cotizar()` | `POST /orquestadorCotizacion` (`CotizacionLogigho: false`) | Solo si el usuario abre "Ver desglose", y solo para roles CEO/Desarrollador |

Ver [cotizacion.service.ts](../core/cotizacion-service.md).

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-09-09 | Iker Acevedo | Extracción del paso 3 a componente propio. Switch con/sin recaudo ahora afecta a las 3 transportadoras. Rediseño de tarjetas con badge de tarifa más económica. Botón de desglose gateado por rol. Labels de "Seguro" unificados. |

---

## Observaciones

- El badge de trayecto (`cubrimiento`) de Envía se evaluó mostrar y se quitó a pedido del usuario — no es un bug, es una decisión de producto. El dato sigue llegando en el response por si se necesita después.
- Ver [reglas-modalidad-pago.ts](../core/reglas-modalidad-pago.md) para cómo el switch de este paso se traduce a "Forma de pago" en el paso 4 (Guía) — la regla es distinta por transportadora y **Envía está invertida**.
