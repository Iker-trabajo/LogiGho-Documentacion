---

## Autor: Adalberto González
Fecha creacion: 2026-06-03
Estado: produccion

# Dominio: Devoluciones

**Namespace:** `SitioLogiGho.views.logistica.guias-devoluciones.models`

---

## ¿Qué representa?

Centraliza todos los contratos de tipo del módulo de Guías de Devoluciones — interfaces de datos, tipos de filtro, constantes de negocio y mensajes entre workers y componente.

---

## Entidad principal: `DevolucionRow`

| Propiedad            | Tipo                                                                          | Descripción                                              |
| ---------------------- | -------------------------------------------------------------------------------- | ----------------------------------------------------------- |
| `_id`                 | `string`                                                                       | Identificador único del documento en MongoDB               |
| `Estado`              | `'devolucion ratificada' \| 'devolucion completada' \| 'devolucion regional' \| 'devolucion'` | Estado normalizado (sin tildes, minúsculas) |
| `Numeropreenvio`      | `string`                                                                       | Número de guía — BigInt del backend, string en frontend    |
| `'Fecha Devolucion'`  | `string`                                                                       | Fecha ISO `YYYY-MM-DD` o `YYYY-MM-DDTHH:mm:ss`             |

### Reglas de negocio

| Regla               | Descripción                                                                              |
| -------------------- | ------------------------------------------------------------------------------------------- |
| Estado normalizado   | Solo los 4 valores de `ESTADOS_VALIDOS`, sin tildes, en minúsculas — normalizado en `parsearYFiltrar()` |
| BigInt como string   | `Numeropreenvio` y `Telefono` se escapan en el JSON crudo antes de parsear                  |

---

## Otras interfaces

### `DevolucionesFiltrosAgg`

Filtros empaquetados que se envían al `agregacion.worker`, construidos por `devoluciones.rules.ts` (`buildFiltrosAgg()`).

### `AggWorkerInput` / `AggWorkerOutput`

Contrato de mensajes del `agregacion.worker` — entrada (`rows`, `tiendas`, `filtros`, `pagina`, `exportarDetalle`) y salida (`chartData`, `tablaResumen`, `tablaDetallePagina`, `tablaDetalleCompleta` opcional).

---

## Relaciones con otros módulos

Ninguna — el módulo no depende de otros dominios del frontend.

---

## Historial de cambios

| Fecha      | Autor              | Cambio                                                                                                             |
| ----------- | -------------------- | -------------------------------------------------------------------------------------------------------------- |
| 2026-07-28 | Adalberto González | Agregados `DevolucionesFiltrosAgg`, `AggWorkerInput/Output`, `ESTADOS_DEVOLUCION`; `Estado` ampliado a 4 valores |

---

## Observaciones

- `ESTADOS_DEVOLUCION` es la fuente única de verdad de los estados consultados al backend — antes duplicada en 3 lugares.
