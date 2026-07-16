## Worker: data-processor.worker

**Autor:** Adalberto González  
**Fecha:** 2026-06-03  
**Estado:** producción  
**Ubicación:** `src/app/views/logistica/guias-devoluciones/workers/data-processor.worker.ts`  
**Tipo:** Web Worker

---

## ¿Qué hace?

Este servicio se encarga de procesar la información recibida desde el servidor.
Una vez recibida la información toma varios bloques de datos comprimidos, los descomprime de forma paralela para acelerar el proceso y devuelve únicamente los datos necesarios.

---

## Mensaje de entrada

```typescript
{ compressedPayloads: string[] }
```

| Campo | Tipo | Descripción |
|---|---|---|
| `compressedPayloads` | `string[]` | Array de strings base64 ZSTD — uno por página del backend |

---

## Mensaje de salida

```typescript
{ processedRows: DevolucionRow[], error?: string }
```

| Campo | Tipo | Descripción |
|---|---|---|
| `processedRows` | `DevolucionRow[]` | Registros descomprimidos y filtrados listos para acumular en memoria |
| `error` | `string?` | Mensaje de error si el procesamiento falló |

---

## Flujo

```
postMessage({ compressedPayloads })
  → inicializar ZSTDDecoder (una sola vez por ciclo de vida del worker)
  → Promise.all(compressedPayloads.map(descomprimirYFiltrar))
      → base64ToUint8Array()   ← worker-utils
      → decoder.decode()       ← ZSTDDecoder
      → textDecoder.decode()   ← worker-utils
      → parsearYFiltrar()      ← worker-utils (sin skipPrefixes)
  → postMessage({ processedRows })
  → worker.terminate()         ← lo hace el componente al recibir la respuesta
```

---

## Ciclo de vida

El componente crea una instancia nueva del worker por cada llamada a `procesarConWorker()` y lo termina (`terminate()`) inmediatamente después de recibir la respuesta. No es un worker persistente.

---

## Endpoints que consume

Ninguno. Recibe los datos ya descargados del componente via `postMessage`.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-06-03 | Adalberto González | Se creo el worker |

---

## Observaciones

- El `ZSTDDecoder` se inicializa una sola vez al recibir el primer mensaje. Si el worker recibe múltiples mensajes  el decoder ya estaría listo.
- Si `compressedPayloads` está vacío o no es un array, el worker responde `{ processedRows: [] }` sin procesar nada.
- Toda la lógica de filtrado y parseo vive en `worker-utils.ts` — este archivo solo orquesta la descompresión.
