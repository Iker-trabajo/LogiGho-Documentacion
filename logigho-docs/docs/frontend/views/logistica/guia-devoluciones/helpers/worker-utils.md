---

## Autor: Adalberto González
Fecha creacion: 2026-06-03
Estado: produccion

# Servicio: worker-utils (funciones compartidas)

**Ubicación:** `src/app/views/logistica/guias-devoluciones/helpers/worker-utils.ts`
**Scope:** Módulo utilitario — se importa directamente en los workers

---

## ¿Qué hace?

Centraliza funciones comunes de los workers: descompresión, conversión de BigInts y normalización de estados.

---

## Métodos

### `base64ToUint8Array(base64: string): Uint8Array`

Convierte base64 a bytes, listo para descomprimir con ZSTD.

### `parsearYFiltrar(jsonString: string, skipPrefixes?: Set<string>): DevolucionRow[]`

Parsea el JSON descomprimido y filtra los registros válidos — omite filas sin fecha, de meses ya cargados, o con estado inválido.

---

## Endpoints que consume

Ninguno. Módulo puro.

---

## Historial de cambios

| Fecha      | Autor              | Cambio                                                        |
| ----------- | -------------------- | ---------------------------------------------------------- |
| 2026-07-28 | Adalberto González | Movido de `workers/` a `helpers/` — sin cambios funcionales |

---

## Observaciones

- `skipPrefixes` es opcional: `data-processor.worker` lo omite, `historico.worker` lo usa para no duplicar meses.
