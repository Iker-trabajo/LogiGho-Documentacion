---

## Autor:
Fecha creacion: 2026-08-20
Estado: produccion

# Servicio: CambioEntregaInterRepository

**Ubicación:** `src/app/views/logistica/cambio-entrega-inter/repository/cambio-entrega-inter.repository.ts`
**Scope:** Módulo `cambio-entrega-inter`

---

## ¿Qué hace?

Encapsula toda la comunicación HTTP del módulo contra el endpoint genérico `/metodoGenerico` (`ApiLambdaCrudGenericoAOT`). No hay una lambda propia para este módulo — todo pasa por el CRUD genérico, igual que `dashboard-sin-despacho` y `resumen-inventario`.

---

## Métodos

### `getCambiosEntrega(): Promise<CambioEntregaInter[]>`

Trae **toda** la colección `AuditoriaCambioEntregaInter`, scopeada por las tiendas asignadas al usuario (`sessionStorage.tiendas_asignadas`).

Sondea `NumeroPaginas` con una primera llamada, y luego dispara 8 páginas en paralelo por lote (`LOTE_PAGINAS = 8`) hasta traer todo — mismo patrón que `ColeccionServicio` del backend, adaptado al front.

| Parámetro | Tipo | Descripción |
| --------- | ---- | ----------- |
| — | — | Sin parámetros — el scoping por tienda se resuelve internamente leyendo `sessionStorage` |

**Retorna:** el array completo, sin paginar — la paginación de la vista es 100% client-side sobre este resultado.

### `getTiendas(): Promise<TiendaRaw[]>`

Trae la colección `Tienda` cruda — el componente raíz la usa para armar el mapa `Ecosistema → Tienda[]` que consume `filtro-tienda-ecosistema` y el selector de ecosistema de `cambio-entrega-analisis`.

### `actualizarGestion(id: string, estado: EstadoGestion): Promise<void>`

`PUT /metodoGenerico?coleccion=AuditoriaCambioEntregaInter&_id={id}` — escribe `EstadoGestion` y `GestionadoPor` (usuario, fecha en hora Colombia, estado resultante).

| Parámetro | Tipo | Descripción |
| --------- | ---- | ----------- |
| `id` | `string` | `_id` del documento en Mongo |
| `estado` | `EstadoGestion` | `'Pendiente'` \| `'Gestionada'` |

Como el backend de la lambda escribe con whitelist de campos (`$setOnInsert` vs `$set`), este `PUT` nunca pisa los campos que escribe la lambda.

---

## Endpoints que consume

| Método | Ruta | Descripción |
| ------ | ---- | ----------- |
| `GET` | `/metodoGenerico?coleccion=AuditoriaCambioEntregaInter&page=N&mcomp=2` | Página N de la colección, comprimida (Zstd) |
| `GET` | `/metodoGenerico?coleccion=Tienda` | Catálogo de tiendas, para armar el mapa Ecosistema→Tienda |
| `PUT` | `/metodoGenerico?coleccion=AuditoriaCambioEntregaInter&_id={id}` | Actualiza `EstadoGestion`/`GestionadoPor` de un documento |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-31 | Iker Acevedo | Repositorio inicial: `getCambiosEntrega`, `getTiendas`, `actualizarGestion`. |

---

## Observaciones

- `fechaUtcMenos5()` (helper interno) arma el timestamp con offset `-05:00` explícito para `GestionadoPor.FechaModificacion` — no usa `Date.toISOString()`, que devuelve UTC.
- No hay caché en memoria del lado del repositorio — el componente raíz decide cuándo recargar (`cargar()`), y las actualizaciones de gestión se aplican optimistamente en el signal `items` sin volver a pedir todo el dataset.
