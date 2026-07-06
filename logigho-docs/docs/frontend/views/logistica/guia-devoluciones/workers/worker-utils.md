---

## Autor: Adalberto González
Fecha creacion: 2026-06-03  
Estado: produccion

# Servicio: worker-utils (funciones compartidas)

**Ubicación:** `src/app/views/logistica/guias-devoluciones/workers/worker-utils.ts`  
**Scope:** Módulo utilitario — no es un servicio Angular, se importa directamente en los workers

---

## ¿Qué hace?

Este módulo reúne las funciones comunes utilizadas por los diferentes servicios encargados de procesar la información. 
Su objetivo es evitar que la misma lógica se implemente varias veces, centralizando tareas como la descompresión de datos, la conversión de valores especiales y la estandarización de los estados de las devoluciones.

---

## Métodos

### `base64ToUint8Array(base64: string): Uint8Array`

Convierte una cadena base64 a un arreglo de bytes `Uint8Array`, listo para descomprimir con ZSTD. No usa `Buffer` de Node — es compatible con el entorno de Web Worker del navegador.

| Parámetro | Tipo | Descripción |
|---|---|---|
| `base64` | `string` | String en base64 recibido del campo `Resultado` del backend |

**Retorna:** `Uint8Array` con los bytes decodificados.

---

### `escapeBigInts(json: string): string`

Escapa en el JSON crudo los campos que el backend envía como números enteros de 16+ dígitos, envolviéndolos en comillas antes de parsear. Aplica a `Numeropreenvio` y `Telefono` (definidos en `BIG_INT_KEYS`).

| Parámetro | Tipo | Descripción |
|---|---|---|
| `json` | `string` | String JSON sin procesar del backend |

**Retorna:** JSON modificado con los BigInts escapados como strings.

---

### `normalizarEstado(estado: string): string`

Normaliza un string de estado a minúsculas sin tildes para comparaciones consistentes.

| Parámetro | Tipo | Descripción |
|---|---|---|
| `estado` | `string` | Estado crudo del documento, ej: `'Devolución Ratificada'` |

**Retorna:** Estado normalizado, ej: `'devolucion ratificada'`.

---

### `parsearYFiltrar(jsonString: string, skipPrefixes?: Set<string>): DevolucionRow[]`

Parsea el JSON descomprimido y filtra los registros válidos. Es la función principal que ambos workers usan después de descomprimir el payload.

| Parámetro | Tipo | Descripción |
|---|---|---|
| `jsonString` | `string` | JSON crudo ya descomprimido |
| `skipPrefixes` | `Set<string>?` | Prefijos `YYYY-MM` a omitir (meses ya en memoria). Solo lo usa `historico.worker`. |

**Retorna:** `DevolucionRow[]` con solo los registros que tienen fecha, estado válido y no pertenecen a meses ya cargados.

**Filtros aplicados internamente:**
1. Omite filas sin `Fecha Devolucion`
2. Omite filas cuyo mes (`YYYY-MM`) esté en `skipPrefixes`
3. Omite filas cuyo estado normalizado no esté en `ESTADOS_VALIDOS`

---

## Constante exportada

| Constante | Tipo | Descripción |
|---|---|---|
| `textDecoder` | `TextDecoder` | Instancia reutilizable de `TextDecoder('utf-8')` — una sola instancia por worker evita crear objetos en cada llamada |

---

## Endpoints que consume

Ninguno. Este módulo es puro — sin HTTP, sin efectos secundarios.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-06-03 | Adalberto González | Se creo el worker |

---

## Observaciones

- `skipPrefixes` es opcional para que `data-processor.worker` pueda llamar `parsearYFiltrar` sin pasar el Set, mientras que `historico.worker` lo pasa para evitar duplicar meses ya cargados.
