---

## Autor: Yadilton Torres
Fecha creacion: 2026-10-14
Estado: desarrollo

# Servicio: PautaRepository

**Ubicación:** `src/app/views/tienda/pauta/repository/pauta.repository.ts`  
**Scope:** `providedIn: 'root'`

---

## ¿Qué hace?

Es el **único** punto del módulo de Pauta que llama al backend. Envuelve el endpoint genérico `/metodoGenerico` (`ApiLambdaCrudGenericoAOT`) con `ConsumoGenericoService`, descomprime las respuestas Zstd con `DecompressionService` y descarga la plantilla desde S3 con `GetObjectService`. No toca la vista ni tiene lógica de negocio.

Lo usan la [Vista Pauta](../pauta.md), el modal [Importacion Pauta](../../../../components/importacion-pauta.md) y [`ValidacionFilasPautaService`](../services/validacion-filas-pauta-service.md).

---

## Métodos

### `consultarPagina(pagina: number, tienda: string): Promise<PaginaPauta>`

Pide una página de la colección `Pauta` comprimida con Zstd (`mcomp: '2'`) y solo con los campos que usa la vista (`campos: Fecha,Idtienda,Tienda,Meta,TikTok,Google,Otros`), para que cada página pese menos.

| Parámetro | Tipo | Descripción |
| --------- | ---- | ----------- |
| `pagina` | `number` | Número de página, empieza en 1 |
| `tienda` | `string` | Tiendas del usuario separadas por coma (`sessionStorage.tiendas_asignadas`) |

**Retorna:** los registros de la página, `numeroPaginas` y `totalRegistros`. La vista usa la página 1 para saber cuántas pedir después (ver `fetchTableData()`).

### `actualizar(id: string, registro: any): Promise<void>`

Actualiza un registro de `Pauta` por su `_id`.

### `insertar(registros: RegistroPauta[]): Promise<void>`

Inserta varios registros de `Pauta` en una sola llamada. Lo usa el modal al importar.

### `consultarTiendas(ids: string[]): Promise<TiendaCatalogo[]>`

Consulta en la colección `Tienda` solo `Id`, `NombreTienda` y `Estado` de todos los Idtienda indicados, en **una sola consulta** (`Id` separados por coma).

| Parámetro | Tipo | Descripción |
| --------- | ---- | ----------- |
| `ids` | `string[]` | Idtienda a verificar |

**Retorna:** los documentos encontrados, o `[]` si la respuesta no es un array. Una tienda puede tener varios documentos (uno por integración), por eso se pide un `lote` de `max(100, ids × 10)`.

### `obtenerPlantilla(): Observable<any>`

Descarga `Pauta.xlsx` del bucket `logigho-plantillas` (contenido en base64).

---

## Endpoints que consume

| Método | Ruta | Descripción |
| ------ | ---- | ----------- |
| `GET` | `/metodoGenerico?coleccion=Pauta&Tienda={tiendas}&mcomp=2&campos=...` (página N) | Página N de la pauta de las tiendas del usuario, comprimida (Zstd) |
| `PUT` | `/metodoGenerico?coleccion=Pauta` (con el `_id`) | Actualiza un registro |
| `POST` | `/metodoGenerico?coleccion=Pauta` | Inserta los registros importados |
| `GET` | `/metodoGenerico?coleccion=Tienda&Id={ids}&campos=Id,NombreTienda,Estado&mcomp=2` | Verifica las tiendas del Excel |
| `GET` | S3 bucket `logigho-plantillas`, key `Pauta.xlsx` (vía `GetObjectService`) | Plantilla de importación |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-10-14 | Yadilton Torres | Repositorio inicial: `consultarPagina`, `actualizar`, `insertar`, `consultarTiendas`, `obtenerPlantilla`. Antes la vista y el modal llamaban directo a `ConsumoGenericoService`. |

---

## Observaciones

- La pauta antes se pedía sin `campos` y con compresión Gzip; ahora va con `mcomp: '2'` (Zstd) y solo los campos de la tabla.
- No hay caché en el repositorio: la vista decide cuándo recargar (al iniciar, después de importar y con `Reintentar`), y el caché de tiendas vive en `ValidacionFilasPautaService` mientras el modal está abierto.
