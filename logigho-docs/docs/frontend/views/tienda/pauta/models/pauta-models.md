---

## Autor: Yadilton Torres
Fecha creacion: 2026-10-14
Estado: desarrollo

# Modelos: pauta.models.ts

**Ubicación:** `src/app/views/tienda/pauta/models/pauta.models.ts`  
**Usado por:** [Vista Pauta](../pauta.md), [Importacion Pauta](../../../../components/importacion-pauta.md), [`PautaRepository`](../repository/pauta-repository.md) y los servicios del módulo

---

## ¿Qué hace?

Reúne en un solo lugar las interfaces y las listas de columnas del módulo de Pauta, para que la vista, el modal, el repositorio y los servicios usen los mismos nombres de campo.

---

## Interfaces

### `RegistroPauta`

Registro tal como se guarda en la colección `Pauta`.

| Campo | Tipo | Descripción |
|---|---|---|
| `Fecha` | `string` | Fecha en `YYYY-MM-DD` |
| `Idtienda` | `number` | ID numérico de la tienda |
| `Tienda` | `string` | Nombre de la tienda |
| `Meta` | `number` | Inversión en Meta Ads |
| `TikTok` | `number` | Inversión en TikTok |
| `Google` | `number` | Inversión en Google |
| `Otros` | `number` | Otras plataformas. En la BD el campo se llama `Otros`, no `Otro` |

### `PaginaPauta`

Una página de la colección `Pauta` ya descomprimida.

| Campo | Tipo | Descripción |
|---|---|---|
| `registros` | `any[]` | Registros de la página |
| `numeroPaginas` | `number` | Cuántas páginas hay en total (`NumeroPaginas` del backend) |
| `totalRegistros` | `number` | Cuántos registros hay en total (`TotalRegistros` del backend) |

### `TiendaCatalogo`

Documento de la colección `Tienda` con los campos que se piden al verificar un Idtienda.

| Campo | Tipo | Descripción |
|---|---|---|
| `Id` | `string \| number` | Idtienda |
| `NombreTienda` | `string` | Nombre real de la tienda |
| `Estado` | `string` | `ACTIVO` o inactiva |

### `FilaPrevia`

Fila del Excel en la previsualización del modal.

| Campo | Tipo | Descripción |
|---|---|---|
| `filaExcel` | `number` | Número de fila real en el Excel (fila 1 = encabezados), para ubicar el error. `0` en las filas nuevas |
| `datos` | `Record<string, string>` | Valor de cada columna como texto, para poder editarlo |
| `errores` | `Record<string, string>` | Mensaje de error por columna. Vacío = fila válida |
| `corregidas` | `Record<string, string>` | Celdas que cambió "Corregir automáticamente" o el nombre de la tienda, con su valor original ("Antes: …") |
| `id` | `number` | Identificador interno de la fila (lo usa `trackBy`) |
| `nueva` | `boolean?` | `true` si la agregó el usuario con "Agregar fila" |

---

## Constantes

| Constante | Valor | Uso |
|---|---|---|
| `COLUMNAS_PAUTA` | `Fecha, Idtienda, Tienda, Meta, TikTok, Google, Otros` | Encabezados exactos de la plantilla y orden de la tabla |
| `COLUMNAS_OBLIGATORIAS_PAUTA` | `Fecha, Idtienda, Tienda, Meta` | No pueden venir vacías (`Tienda` se llena sola con el Idtienda) |
| `COLUMNAS_NUMERICAS_PAUTA` | `Idtienda, Meta, TikTok, Google, Otros` | Solo números enteros: se alinean a la derecha y usan el teclado numérico |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-10-14 | Yadilton Torres | Modelos iniciales del módulo de Pauta |
