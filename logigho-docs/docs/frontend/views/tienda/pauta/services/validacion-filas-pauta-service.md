---

## Autor: Yadilton Torres
Fecha creacion: 2026-10-14
Estado: desarrollo

# Servicio: ValidacionFilasPautaService

**Ubicación:** `src/app/views/tienda/pauta/services/validacion-filas-pauta.service.ts`  
**Scope:** `providers` del modal `ImportacionPautaComponent`

---

## ¿Qué hace?

Valida cada fila de la previsualización del modal [Importacion Pauta](../../../../components/importacion-pauta.md) y verifica sus tiendas contra la colección `Tienda`. Guarda qué tiendas ya se consultaron, por eso se registra en `providers` del modal: vive mientras vive el modal y no se comparte con otras pantallas. No lee el Excel ni toca la vista; las consultas las hace [`PautaRepository`](../repository/pauta-repository.md).

---

## Estado interno

| Propiedad | Tipo | Descripción |
|---|---|---|
| `tiendasPorId` *(privado)* | `Map<string, { nombre, activa } \| null>` | Tiendas ya consultadas. `null` = ese Id no existe |
| `idsBuscando` | `Set<string>` | Idtienda que se están consultando en este momento |
| `errorTiendas` | `boolean` | Falló la consulta de tiendas; mientras sea `true` no se puede importar |
| `tiendasPermitidas` *(privado)* | `string[] \| null` | Tiendas del usuario en minúscula; `null` si tiene `Todas` |

---

## Métodos

### `prepararNuevoArchivo(): void`

Al cargar un archivo quita el error de tiendas anterior y vuelve a leer las tiendas del usuario de `sessionStorage.tiendas_asignadas`.

### `validarFila(fila: FilaPrevia): void`

Valida una fila y deja sus errores en `fila.errores`. Se llama al cargar el archivo, al editar una celda y cuando llega la respuesta de tiendas.

**Proceso:**
1. `Fecha`, `Idtienda` y `Meta` son obligatorios; `TikTok`, `Google` y `Otros` pueden ir vacíos (se importan como `0`). `Tienda` no se valida porque se llena sola.
2. `Idtienda` y los montos solo aceptan dígitos (`Solo números`); la `Fecha` debe ser una fecha real en `YYYY-MM-DD` (rechaza, por ejemplo, `2026-02-30`).
3. Si el `Idtienda` es válido, lo revisa contra el catálogo con `validarTienda()`.

### `validarTienda(fila)` *(privado)*

1. Si la tienda todavía se está consultando o no se ha consultado, no marca error (el botón Importar espera).
2. Devuelve `Tienda no existe`, `Tienda inactiva` o `No es tu tienda` según el caso; `Sin verificar` si falló la consulta.
3. Pone el nombre real en la columna `Tienda`. Si el Excel traía un nombre distinto, guarda el original en `fila.corregidas` para mostrar "Antes: …".

### `idsSinConsultar(ids: string[]): string[]`

De los Idtienda recibidos devuelve los que son números válidos y todavía no se han consultado ni se están consultando (sin repetidos).

### `consultarTiendas(pendientes: string[]): Promise<void>`

Consulta los Idtienda (ya filtrados con `idsSinConsultar`).

**Proceso:**
1. Los marca como "buscando" **antes** de la primera espera, para que el modal pueda revalidar enseguida y mostrar "Buscando…".
2. Pide `Id`, `NombreTienda` y `Estado` de todos en una sola consulta. Si una tienda tiene varios documentos, cuenta como activa si alguno está `ACTIVO`.
3. Los Id que no vienen en la respuesta quedan marcados como inexistentes.
4. Si la consulta falla, marca `errorTiendas`. **Nunca lanza error.**

### `hayTiendasPendientes(filas: FilaPrevia[]): boolean`

`true` si hay filas cuyo Idtienda todavía no se ha verificado (consultando o sin consultar). Mientras sea `true` no se puede importar.

### `corregirSeparadores(valor: string): string | null`

Quita todo lo que no sea dígito: `"1.500.000"` → `"1500000"`, `"1500.5"` → `"15005"` (no se usan decimales). Los valores con letras no se corrigen porque no se sabe qué número quisieron poner.

**Retorna:** el valor corregido, o `null` si no se puede corregir automáticamente.

---

## Endpoints que consume

| Método | Ruta | Descripción |
| ------ | ---- | ----------- |
| `GET` | `/metodoGenerico?coleccion=Tienda&Id={ids}&campos=Id,NombreTienda,Estado&mcomp=2` | Vía `PautaRepository.consultarTiendas()` |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-10-14 | Yadilton Torres | Servicio inicial: validación por celda, verificación de tiendas y corrección de separadores, separadas del modal. 7 pruebas. |

---

## Observaciones

- **"No es tu tienda" es solo de frontend** y compara por el **nombre** de la tienda en minúscula, igual que el resto de módulos que usan `tiendas_asignadas`. La validación real debe ir en el backend; también se podría comparar por Id usando `sessionStorage.tiendas_id`.
- Las tiendas inactivas bloquean la importación: para cargarles pauta primero hay que activarlas en Administración de tiendas.
