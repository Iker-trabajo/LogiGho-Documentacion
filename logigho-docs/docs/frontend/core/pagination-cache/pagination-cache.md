---

## Autor: Adalberto González
Fecha creacion: 2026-06-06
Estado: produccion

# Servicio: PaginationCacheService

**Ubicación:** `src/app/core/application/services/pagination-cache/pagination-cache.service.ts`  
**Scope:** `providedIn: 'root'`

---

## ¿Qué hace?

Este servicio almacena temporalmente en el navegador la información ya procesada de los módulos que manejan grandes volúmenes de datos.
Gracias a esto, cuando un usuario vuelve a consultar la misma información en poco tiempo, la aplicación puede recuperarla desde el almacenamiento local en lugar de solicitarla nuevamente al servidor.

---

## Métodos

### `getCachedPage(key: string): Promise<{ data: any[]; totalRegistros: number } | null>`

Busca una entrada en caché por clave. Si expiró la elimina y retorna `null`.

| Parámetro | Tipo     | Descripción              |
| --------- | -------- | ------------------------ |
| `key`     | `string` | Identificador del caché  |

**Retorna:** Objeto con `data` y `totalRegistros`, o `null` si no existe o expiró.

---

### `setCachedPage(key: string, data: any[], totalRegistros: number): Promise<void>`

Guarda datos procesados en caché con timestamp. Si se supera el límite de 100 entradas, elimina la más antigua.

| Parámetro        | Tipo     | Descripción                        |
| ---------------- | -------- | ---------------------------------- |
| `key`            | `string` | Identificador del caché            |
| `data`           | `any[]`  | Datos ya procesados por el Worker  |
| `totalRegistros` | `number` | Total real de registros en BD      |

---

### `clearCache(): Promise<void>`

Elimina todas las entradas del store. Se invoca en el logout.

---

## Historial de cambios

| Fecha      | Autor              | Cambio |
|------------|--------------------|--------|
| 2026-06-06 | Adalberto González | Se reemplazó el TTL por sesión (Cognito token) por un TTL fijo de 3 minutos. Se cambió el valor almacenado de datos comprimidos a datos ya procesados por el Worker. Se agregó `totalRegistros` al schema. Se subió la versión de IndexedDB a 2 para migrar el store limpio. |

---

## Observaciones
