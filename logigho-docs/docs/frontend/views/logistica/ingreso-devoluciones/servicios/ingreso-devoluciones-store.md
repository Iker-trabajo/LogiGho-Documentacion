## Autor: Iker Acevedo

Fecha creacion: 2026-08-26

Estado: produccion

## Servicio: IngresoDevolucionesStore

**Ubicación:** `helpers/ingreso-devoluciones.store.ts`

---

## ¿Qué hace?

Único dueño del estado compartido del módulo, con Angular signals. No habla HTTP, no contiene lógica de negocio — solo guarda lo que el orquestador (`ingreso-devoluciones.component.ts`) le indica y lo expone de forma readonly a los componentes hijos.

| Signal | Tipo | Qué guarda |
| ------ | ---- | ---------- |
| `loteActivo` | `EstadoLoteResponse \| null` | El lote de archivo grande en curso o recién terminado |
| `polling` | `boolean` | Si el ciclo de polling está corriendo |
| `loading` | `boolean` | Mientras se llama a `iniciarLote` |
| `error` | `string \| null` | Mensaje de error visible en toda la vista |
| `historial` | `LoteHistorial[]` | Tabla de lotes cerrados |
| `historialLoading` | `boolean` | — |
| `detalleRechazadas` | `GuiaProcesada[]` | Del lote activo, tras terminar |
| `detalleResueltas` | `InventarioDevolucionDoc[]` | Del lote activo, tras terminar |

### Computed

```typescript
loteEnCurso   = () => loteActivo()?.Estado in ['Pendiente', 'EnProceso']
loteTerminado = () => loteActivo()?.Estado in ['Completado', 'Fallido']
```

Derivados siempre del mismo signal fuente (`loteActivo`), nunca seteados a mano en paralelo — si se setearan como flags independientes, tarde o temprano quedarían desincronizados (se actualiza uno y se olvida el otro). Un `computed` garantiza que nunca mientan.

---

## Por qué NO corre el polling acá

El store nunca hace `setInterval` ni llama al repositorio — esa responsabilidad vive en `ingreso-devoluciones.component.ts`. Si el polling viviera en el store (un singleton `providedIn: 'root'`, que vive para siempre en la aplicación), sería imposible destruirlo limpio cuando el componente que lo necesita se destruye.

---

## `reset()` — qué limpia y qué no

```typescript
reset(): void {
    loteActivo = null; polling = false; loading = false; error = null;
    detalleRechazadas = []; detalleResueltas = [];
    // NO toca historial ni historialLoading — eso se recarga aparte
}
```
