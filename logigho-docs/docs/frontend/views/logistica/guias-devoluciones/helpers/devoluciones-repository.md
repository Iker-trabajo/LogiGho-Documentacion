## Servicio: DevolucionesRepository

**Autor:** Adalberto González
**Fecha:** 2026-07-28
**Estado:** producción
**Ubicación:** `src/app/views/logistica/guias-devoluciones/helpers/devoluciones.repository.ts`
**Scope:** `providedIn: 'root'`

---

## ¿Qué hace?

Centraliza el acceso HTTP del módulo — URLs, filtros de fecha/tienda y la config del `historico.worker`. Antes esta lógica vivía inline en el componente.

---

## Métodos

### `getPrimeraPagina(estado: string)`

Trae la página 1 de un estado con el filtro de fecha rápido (últimos 60 días).

### `getTiendas()`

Trae las tiendas activas asignadas al usuario.

### `buildHistoricoWorkerConfig(): HistoricoWorkerConfig`

Construye la config para los 2 workers de histórico, con `headerSecurity` correctamente nombrado (antes `headersecurity`, bug corregido) y ventana de 4 meses.

### `buildUrl(estado, fechasFiltro?)`

Construye la URL de `PedidosInter`, incluyendo `&campos=` para reducir el peso de cada página de ~951 KB a ~317 KB.

---

## Endpoints que consume

| Método | Ruta | Descripción |
|---|---|---|
| `GET` | `metodoGenerico?coleccion=PedidosInter&Tienda=...&Estado=...&fechasFiltro=...&campos=...&mcomp=2&page=N` | Fases 1 y 2 |
| `GET` | `metodoGenerico?coleccion=Tienda&Estado=ACTIVO&mcomp=1` | Tiendas activas |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-28 | Adalberto González | Creación, extrayendo HTTP inline del componente. Bug `headersecurity`/`headerSecurity` corregido. Histórico reducido a 4 meses. Agregado `&campos=` |

---

## Observaciones

- Se evaluó el parámetro `pipeline` de MongoDB para consolidar las 7 peticiones por estado en una sola consulta — causaba timeout en API Gateway y se descartó.
