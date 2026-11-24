## Autor: Yadilton Torres

Fecha creacion: 2026-10-30

Estado: desarrollo

## Servicio: NovedadesStore

**Ubicación:** `helpers/novedades.store.ts`

---

## ¿Qué hace?

Guarda en signals las novedades cargadas y el estado de cada carga. Llama al repositorio, pero no contiene reglas de negocio (esas están en `rules.ts`) ni lógica de filtros (vive en el componente).

**Se provee en la vista** (`providers: [NovedadesStore]`), no en `root`: hay un store por visita, y al salir de la pantalla Angular lo destruye con todo lo que tenía en memoria.

| Signal | Tipo | Qué guarda |
| ------ | ---- | ---------- |
| `filas` | `NovedadFila[]` | Todas las novedades cargadas (Inter + Envía/Servientrega), más recientes primero |
| `estado` | `'cargando' \| 'error' \| 'listo'` | Estado de la carga principal |
| `cargandoCerradas`, `cerradasCargadas` | `boolean` | Las cerradas se piden aparte; mientras no lleguen, las pestañas muestran "…" |
| `detalleCargando` | `string \| null` | Guía cuyo detalle se está pidiendo |
| `fotos` | `Map<s3Key, FotoEvidencia>` | Fotos de la novedad ya pedidas |
| `fuentesFallidas` | `string[]` | Fuentes que fallaron (aviso de carga parcial) |
| `topeAlcanzado` | `string[]` | Consultas que llegaron a `LIMITE_CONSULTA` |
| `ahora` | `computed` | Hora de la última carga: con ella se cuentan los días abiertos, para que no cambien con el reloj mientras se mira |

---

## `cargar()` — dos fuentes independientes

```typescript
const [inter, pedidos] = await Promise.allSettled([repo.listarInter(null), repo.listarPedidos()]);
```

Si una falla, se muestra la otra y se avisa cuál faltó. Solo es error total si fallan las dos. Cada carga **reemplaza** lo anterior y descarta el detalle y las fotos guardados.

Un contador de **turno** evita que una respuesta vieja escriba después de una recarga o de salir de la vista.

---

## Métodos

| Método | Qué hace |
| ------ | -------- |
| `asegurarCerradas()` | Pide una sola vez las cerradas de Inter de los últimos `DIAS_CERRADAS` (7) días, al abrir Cerradas/Todas |
| `detalle(fila)` | Pide el detalle de una guía de Inter la primera vez y lo deja guardado en `filas`; si falla, devuelve la fila como estaba |
| `cargarFotos(evidencias)` | Pide a S3 cada foto que aún no se tenga; si falla, la marca con error y deja el motivo en la consola |

---

## Memoria

- `ngOnDestroy()` sube el turno y suelta filas y fotos de inmediato.
- Nada se guarda en el navegador: ni sessionStorage, ni localStorage, ni IndexedDB.
