---

## Autor: Adalberto González
Fecha creacion: 2026-06-03
Estado: produccion

# Servicio: data-processor.worker

**Ubicación:** `src/app/views/logistica/guias-devoluciones/helpers/data-processor.worker.ts`
**Scope:** Web Worker (instancia nueva por llamada, no persistente)

---

## ¿Qué hace?

Descomprime en paralelo varios payloads ZSTD recibidos de las fases 1/2 de carga y devuelve las filas ya filtradas.

---

## Métodos

### `onmessage({compressedPayloads: string[]}): void`

| Parámetro            | Tipo       | Descripción                                    |
| ---------------------- | ----------- | ------------------------------------------------- |
| `compressedPayloads`  | `string[]` | Payloads base64 ZSTD, uno por página del backend  |

**Retorna:** `{processedRows: DevolucionRow[], error?: string}`.

---

## Endpoints que consume

Ninguno. Recibe los datos ya descargados por el componente.

---

## Historial de cambios

| Fecha      | Autor              | Cambio                                                              |
| ----------- | -------------------- | ---------------------------------------------------------------- |
| 2026-07-28 | Adalberto González | Movido de `workers/` a `helpers/` — sin cambios funcionales       |

---

## Observaciones

- El componente crea una instancia nueva por cada llamada a `procesarConWorker()` y la termina al recibir respuesta.
