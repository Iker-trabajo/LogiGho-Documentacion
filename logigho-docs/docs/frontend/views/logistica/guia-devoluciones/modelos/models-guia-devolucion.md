---

## Autor: Adalberto González
Fecha creacion: 2026-06-03
Estado: produccion

# Dominio: Devoluciones

**Namespace:** `SitioLogiGho.views.logistica.guias-devoluciones.models`

---

## ¿Qué representa?

Centraliza todos los contratos de tipo del módulo de Guías de Devoluciones. Define las interfaces de datos, los tipos de filtro, las constantes de negocio y los mensajes entre workers y componente. Es la única fuente de verdad de tipos para toda la capa del módulo.

---

## Entidad principal: `DevolucionRow`

| Propiedad | Tipo | Descripción |
|---|---|---|
| `_id` | `string` | Identificador único del documento en MongoDB |
| `Estado` | `'devolucion ratificada' \| 'devolucion completada'` | Estado normalizado (sin tildes, minúsculas) |
| `Tienda` | `string` | Nombre de la tienda origen de la devolución |
| `Transportadora` | `string` | Transportadora que maneja la guía |
| `Numeropreenvio` | `string` | Número de guía — viene como BigInt del backend, se almacena como string |
| `'Fecha Devolucion'` | `string` | Fecha en formato ISO `YYYY-MM-DD` o `YYYY-MM-DDTHH:mm:ss` |
| `Telefono` | `string?` | Teléfono del destinatario — BigInt en backend, string en frontend |
| `Departamento` | `string?` | Departamento de destino |
| `Observaciones` | `string?` | Texto libre de observaciones de la devolución |
| `ValorDeclarado` | `number?` | Valor en COP declarado para la guía |

### Reglas de negocio

| Regla | Descripción |
|---|---|
| Estado normalizado | Solo se aceptan `'devolucion ratificada'` y `'devolucion completada'` — sin tildes, en minúsculas. La normalización ocurre en `parsearYFiltrar()` antes de guardar en memoria. |
| BigInt como string | `Numeropreenvio` y `Telefono` superan los 15 dígitos. Se escapan en el JSON crudo antes de parsear para evitar pérdida de precisión. |

---

## Otras interfaces

### `TiendaInfo`

Metadato de tienda proveniente de la colección `Tienda` en MongoDB.

| Propiedad | Tipo | Descripción |
|---|---|---|
| `NombreTienda` | `string` | Nombre de la tienda |
| `Ecosistema` | `string` | Agrupación comercial a la que pertenece |
| `Transportadora` | `string` | Transportadora asignada |
| `Estado` | `string` | Estado de la tienda (ej: `ACTIVO`) |

### `DayPoint`

Punto de datos para una barra del chart de barras apiladas.

| Propiedad | Tipo | Descripción |
|---|---|---|
| `dateKey` | `string` | Clave de ordenamiento `YYYY-MM-DD` |
| `dateLabel` | `string` | Etiqueta visual para el eje X, ej: `may 15` |
| `recibida` | `number` | Cantidad de guías con estado completado |
| `pendiente` | `number` | Cantidad de guías con estado ratificado |

### `TablaFila`

Nodo del árbol jerárquico: mes → fecha → tienda → guía.

| Propiedad | Tipo | Descripción |
|---|---|---|
| `key` | `string` | Clave única del nodo para trackBy |
| `label` | `string` | Texto visible en la tabla |
| `recibida` | `number` | Total acumulado de guías recibidas |
| `pendiente` | `number` | Total acumulado de guías pendientes |
| `hijos` | `TablaFila[]?` | Nodos hijos del nivel siguiente |
| `abierto` | `boolean` | Controla si el nodo está expandido en la UI |

### `FilterState`

Estado completo de un filtro multi-select.

| Propiedad | Tipo | Descripción |
|---|---|---|
| `key` | `FilterKey` | Identificador único del filtro |
| `label` | `string` | Texto visible en la UI |
| `options` | `string[]` | Opciones disponibles para seleccionar |
| `selected` | `string[]` | Opciones actualmente seleccionadas |
| `search` | `string` | Texto de búsqueda dentro del dropdown |
| `open` | `boolean` | Si el dropdown está abierto |

### `FilterKey`

Union type con las 7 claves válidas de filtro: `'mes' | 'fecha' | 'tipoDia' | 'tienda' | 'ecosistema' | 'transp' | 'estado'`

### `WorkerBatchMessage`

Mensaje que emite `historico.worker` por cada lote procesado.

| Propiedad | Tipo | Descripción |
|---|---|---|
| `rows` | `DevolucionRow[]` | Registros del lote. Array vacío en el mensaje final. |
| `done` | `boolean` | `true` solo en el último mensaje — señal de finalización |
| `error` | `string?` | Mensaje de error si el lote falló |

### `BackendResponse`

Forma de la respuesta HTTP del backend comprimido.

| Propiedad | Tipo | Descripción |
|---|---|---|
| `Resultado` | `string` | Payload en base64 comprimido con ZSTD |
| `NumeroPaginas` | `number?` | Total de páginas disponibles para paginar |

---

## Constantes

| Constante | Tipo | Descripción |
|---|---|---|
| `ESTADOS_VALIDOS` | `Set<string>` | Estados normalizados aceptados: `devolucion ratificada`, `devolucion completada` |
| `BIG_INT_KEYS` | `readonly string[]` | Campos que el backend envía como BigInt: `Numeropreenvio`, `Telefono` |
| `ESTADO_OPTIONS` | `readonly string[]` | Opciones de display para el filtro de estado |
| `MESES_CORTO` | `readonly string[]` | Nombres cortos de meses en español: `ene, feb, ...` |
| `MESES_LARGO` | `readonly string[]` | Nombres completos de meses en español: `Enero, Febrero, ...` |
| `DIAS_SEMANA` | `readonly string[]` | Días de la semana en español comenzando por `Domingo` |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-06-03 | Adalberto González | Se crearon los modelos |

---

## Observaciones