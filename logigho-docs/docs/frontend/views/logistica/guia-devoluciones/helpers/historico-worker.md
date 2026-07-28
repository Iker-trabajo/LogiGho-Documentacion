---

## Autor: Adalberto González
Fecha creacion: 2026-06-03
Estado: produccion

# Servicio: historico.worker

**Ubicación:** `src/app/views/logistica/guias-devoluciones/helpers/historico.worker.ts`
**Scope:** Web Worker

---

## ¿Qué hace?

Obtiene el historial de devoluciones acotado a los últimos 4 meses, consultando todas las páginas disponibles y entregando los registros a medida que los recibe.

---

## Métodos

### `onmessage(HistoricoWorkerConfig): void`

Único punto de entrada.

| Parámetro        | Tipo       | Descripción                                                |
| ----------------- | ----------- | -------------------------------------------------------------- |
| `headerSecurity`  | `string`   | Header de seguridad (antes `headersecurity`, bug corregido)    |
| `fechasFiltro`    | `string`   | Rango `YYYYMMDD-YYYYMMDD` — acota a los últimos 4 meses        |
| `mesesYaCargados` | `string[]` | Prefijos `YYYY-MM` ya en memoria, se omiten                    |

**Retorna:** emite `WorkerBatchMessage` N veces, con `done: true` en el mensaje final.

---

## Endpoints que consume

| Método | Ruta                                                                                    | Descripción                          |
| ------- | ------------------------------------------------------------------------------------------ | --------------------------------------- |
| `GET`  | `{apiUrl}metodoGenerico?coleccion=PedidosInter&Estado=..&fechasFiltro=..&campos=..&page=N` | Página N del histórico (4 meses)      |

---

## Historial de cambios

| Fecha      | Autor              | Cambio                                                                                                              |
| ----------- | -------------------- | ------------------------------------------------------------------------------------------------------------------ |
| 2026-07-28 | Adalberto González | Ventana reducida a 4 meses; corregido bug `headersecurity`/`headerSecurity`; agregado `&campos=`; typo `fechPagina`→`fetchPagina` |

---

## Observaciones

- El componente lanza 2 instancias en paralelo, según `ESTADOS_HISTORICO_WORKER_1`/resto.
