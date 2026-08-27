## Autor: Iker Acevedo

Fecha creacion: 2026-08-26

Estado: produccion

## Componente: historial-lotes

**Selector:** `app-historial-lotes`
**Ubicación:** `components/historial-lotes/`

---

## ¿Qué hace?

Tabla de lotes procesados. Cada fila se despliega **hacia abajo** para mostrar su [`detalle-lote`](detalle-lote.md), sin sacar al operario de la lista ni navegar a otra pantalla — un acordeón de a una fila a la vez (`expandido: string | null`).

```
@Output() reintentar: EventEmitter<string[]>    lo atiende el shell, llama a iniciarLote de nuevo
```

---

## El buscador de arriba es GLOBAL

```typescript
lotesOrdenados  — filtra el HISTORIAL COMPLETO por textoBusqueda, no solo el lote abierto
loteCoincide(lote, texto)
  idLote.includes(texto)                          siempre
  detalleRechazadas.some(g => g.Guia.includes(texto))    siempre — ya viene en el documento
  aceptadasPorLote.get(lote.jobId)?.some(...)      SOLO si ese lote ya se abrio antes (cache)
```

Escribir un número de guía arriba filtra los **lotes** que la contienen (no las filas de un detalle abierto). Al abrir uno de esos lotes, ese mismo texto baja como `filtroGuiaInicial` al `detalle-lote`, que lo posiciona en la pestaña correcta.

⚠️ **Limitación honesta, mostrada en el estado vacío del buscador:** las guías **aceptadas** solo se pueden encontrar por este buscador en lotes que el operario **ya abrió al menos una vez** — por la carga perezosa de `aceptadasPorLote` (ver abajo). Las rechazadas siempre se encuentran, sin importar si el lote se abrió, porque ya vienen completas en `LoteHistorial.detalleRechazadas`.

---

## Carga perezosa de aceptadas

```typescript
private aceptadasPorLote = new Map<string, InventarioDevolucionDoc[]>();

alternar(lote)
  si ya esta en el Map -> no vuelve a pedir nada
  si no -> cargarAceptadas(lote)
             repo.obtenerInventarioPorJobId(lote.jobId)     correlacion exacta, ver ADR-003
             filtra Validacion === 'OK'
```

Traer las guías aceptadas de **todos** los lotes del historial de entrada sería pesado y la mayoría nunca se abren — se piden solo cuando el operario de verdad despliega esa fila, y se cachean para no repetir la consulta si la vuelve a abrir.

⚠️ **Lotes viejos sin `JobId`:** los procesados antes de que el backend empezara a escribir `JobId` en cada fila de `InventarioDevolucion` no tienen ese campo — sus lotes muestran la pestaña de "Aceptadas" vacía. Es solo el histórico previo al cambio de [ADR-003](../../../../../backend/lambdas-dotnet/aplicacion/Logigho/ApiLambdaDevolucionesMasivo/adr/ADR-003-jobid-en-inventariodevolucion.md), no un bug.

---

## Estado en lenguaje de operario: "Completado con alertas"

```typescript
etiquetaEstado(lote)
  Completado && rechazadas > 0  -> "Completado con alertas"
  Completado && rechazadas == 0 -> "Completado"
  EnProceso                     -> "En proceso"
  Fallido                       -> "Fallido"
```

Para el backend, un job con rechazos y uno sin rechazos son ambos `Estado == "Completado"` — no hay diferencia en el dominio. Pero para quien tiene las cajas físicas en la mano, sí importa saber de un vistazo si hay algo que revisar. `claseEstado` deriva el color de la pastilla con la misma regla, para que el color y el texto nunca digan cosas distintas.

---

## Métodos para el tour guiado

```typescript
expandirPrimero()   despliega el primer lote de la lista — usado por ingreso-devoluciones.component.ts
                     para que el paso del tour que explica "Rechazadas" tenga algo que iluminar
cerrarTodos()        vuelve al estado colapsado
```

El shell (`ingreso-devoluciones.component.ts`) accede a estos métodos vía `@ViewChild(HistorialLotesComponent)` — se decidió exponerlos en vez de subir el estado `expandido` al componente padre, porque ese estado es asunto interno del historial y el shell no lo necesita para nada más que el tour.
