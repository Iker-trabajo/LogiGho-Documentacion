---

## Autor: Adalberto González
Fecha creacion: 2026-06-03  
Estado: desarrollo

# Worker: historico.worker

**Ubicación:** `src/app/views/logistica/guias-devoluciones/workers/historico.worker.ts`  
**Tipo:** Web Worker

---

## ¿Qué hace?

Este servicio se encarga de obtener el historial completo de devoluciones.
Para hacerlo, consulta automáticamente todas las páginas de información disponibles en el servidor y va entregando los registros a medida que los recibe, sin esperar a que termine de descargar todo el histórico.

---

## Mensaje de entrada (`HistoricoWorkerConfig`)

| Campo | Tipo | Descripción |
|---|---|---|
| `apiUrl` | `string` | URL base del API Gateway |
| `token` | `string` | JWT del usuario en sessionStorage |
| `headersecurity` | `string` | Header de seguridad requerido por el backend |
| `estados` | `string[]` | Estados a consultar, ej: `['Devolución ratificada']` |
| `tiendaParam` | `string` | Tiendas asignadas al usuario para filtrar. Vacío = todas |
| `mesesYaCargados` | `string[]` | Prefijos `YYYY-MM` ya en memoria — el worker los salta para no duplicar |

---

## Mensaje de salida (`WorkerBatchMessage`)

Se emite **N veces** — una por cada lote procesado, más un mensaje final con `done: true`.

| Campo | Tipo | Descripción |
|---|---|---|
| `rows` | `DevolucionRow[]` | Registros del lote. Array vacío en el mensaje final. |
| `done` | `boolean` | `true` solo en el último mensaje — señal de finalización |
| `error` | `string?` | Presente si un lote individual falló (el worker continúa con el siguiente) |

---

## Flujo

```
postMessage(HistoricoWorkerConfig)
  → inicializar ZSTDDecoder
  → para cada estado en estados[]:
      → buildUrl(apiUrl, estado, tiendaParam)
      → fetchPagina(url, página 1)          ← HTTP GET con headers de auth
          → descomprimirLote(Resultado, skipPrefixes)
              → base64ToUint8Array()         ← worker-utils
              → decoder.decode()             ← ZSTDDecoder
              → parsearYFiltrar(skipPrefixes) ← worker-utils
          → emitirLote(rows, done: false)    → postMessage al componente
      → para páginas 2..N en lotes de CONCURRENT_PAGES=2:
          → Promise.all(fetchPagina x2)
          → descomprimirLote por cada página
          → emitirLote(rows, done: false)
  → emitirLote([], done: true)              ← señal final
```

---

## Ciclo de vida

El componente lanza **2 instancias** de este worker en paralelo:
- **Worker 1:** estados `['Devolución ratificada']`
- **Worker 2:** estados `['Devolucion Completada', 'Devolución completada']`

El componente cuenta cuántos workers emiten `done: true`. Cuando los 2 terminan, activa `historicoListo = true` y detiene la animación de carga. Cada worker se termina con `worker.terminate()` al recibir `done: true`.

---

## Endpoints que consume

| Método | Ruta | Descripción |
|---|---|---|
| `GET` | `{apiUrl}metodoGenerico?coleccion=PedidosInter&Estado={estado}&Tienda={tienda}&mcomp=2&page={n}` | Obtiene página N de devoluciones comprimidas con ZSTD |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-06-03 | Adalberto González | Se creo el worker |

---

## Observaciones

- `CONCURRENT_PAGES = 2` — se descargan 2 páginas en paralelo por estado para no saturar el backend. Ajustar con cuidado.
- Si una página individual falla, el error se loguea pero el worker continúa con las demás páginas — no aborta todo el histórico.
