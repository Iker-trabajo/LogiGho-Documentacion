---

## Autor: Adalberto González
Fecha creacion: 2026-07-28
Estado: produccion

# Servicio: DevolucionesRepository

**Ubicación:** `src/app/views/logistica/guias-devoluciones/helpers/devoluciones.repository.ts`
**Scope:** `providedIn: 'root'`

---

## ¿Qué hace?

Centraliza el acceso HTTP del módulo — URLs, filtros de fecha/tienda y la config del `historico.worker`. Antes vivía inline en el componente.

---

## Métodos

### `getPrimeraPagina(estado: string): Promise<{resultado, totalPaginas, url}>`

Trae la página 1 de un estado, ventana de últimos 60 días.

### `getPagina(url: string, page: number): Promise<string>`

Trae una página específica dado un `url` ya construido.

### `getTiendas(): Promise<TiendaInfo[]>`

Trae las tiendas activas asignadas al usuario.

### `buildHistoricoWorkerConfig(): HistoricoWorkerConfig`

Construye la config para los 2 workers de histórico (ventana: 4 meses).

**Retorna:** incluye `headerSecurity` con el casing correcto (antes `headersecurity`, bug corregido en esta migración).

---

## Endpoints que consume

| Método | Ruta                                                                                          | Descripción                          |
| ------- | ----------------------------------------------------------------------------------------------- | --------------------------------------- |
| `GET`  | `metodoGenerico?coleccion=PedidosInter&Tienda=..&Estado=..&fechasFiltro=..&campos=..&mcomp=2&page=N` | Fases 1 y 2 (últimos 60 días)         |
| `GET`  | `metodoGenerico?coleccion=Tienda&Estado=ACTIVO&mcomp=1`                                         | Tiendas activas                       |

---

## Historial de cambios

| Fecha      | Autor              | Cambio                                                                                                    |
| ----------- | -------------------- | ------------------------------------------------------------------------------------------------------- |
| 2026-07-28 | Adalberto González | Creación del Repository, extrayendo HTTP del componente; corregido bug `headersecurity`/`headerSecurity`; histórico reducido a 4 meses; agregado `&campos=` |

---

## Observaciones

- Se evaluó el parámetro `pipeline` de MongoDB para consolidar las 7 peticiones por estado en una sola consulta — causaba timeout en API Gateway y se descartó.
