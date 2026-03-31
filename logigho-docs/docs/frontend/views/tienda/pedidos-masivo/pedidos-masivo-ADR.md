---
autor: Iker
fecha_creacion: 2026-03-31
estado: aceptada
---

# Mover el parseo del Excel a un Web Worker

**Autor:** Iker Acevedo

**Fecha:** 2026-03-31

**Estado:** Aceptada

---

## Contexto

El módulo de carga masiva de guías (`CargaGuiasComponent`) permite subir un archivo `.xlsx` con hasta **5 000 filas** de pedidos. El flujo original ejecutaba todo el procesamiento del archivo directamente en el hilo principal del navegador:

1. Parseo del Excel con **SheetJS** (`xlsx`) → array 2D de celdas
2. Conversión a JSON con `convertirExcelAJson()` — eliminando filas vacías y calculando una clave de deduplicación por cada fila concatenando los valores de ~80 columnas con `|`
3. Agrupación de pedidos por tienda con `agruparPorTienda()`

Estas tres operaciones son **CPU-bound y síncronas**: mientras se ejecutan, el event loop de JavaScript queda bloqueado. En archivos grandes (2 000–5 000 filas) esto producía que:

- La visual se congelaba visualmente durante varios segundos
- El spinner de "Procesando archivo..." no se animaba (bloqueado por el mismo hilo que lo debería animar)
- En dispositivos lentos el navegador mostraba el diálogo "Esta página no responde"
- Los clicks del usuario durante ese período se acumulaban y se ejecutaban después del desbloqueo

Restricción adicional: SheetJS es una librería pesada. Importarla en el bundle principal aumenta el tiempo de carga inicial de la app para todos los módulos, aunque solo `CargaGuiasComponent` la usa.

---

## Opciones consideradas

### Opción A — Mantener el procesamiento en el hilo principal 

Conservar `convertirExcelAJson()` y `agruparPorTienda()` tal como estaban: ejecutados síncronamente en el componente Angular inmediatamente después de leer el `ArrayBuffer` con `FileReader`.

**Pros:**

- Sin complejidad adicional: todo el código en un mismo archivo
- Sin necesidad de serializar/deserializar datos entre hilos
- Angular no requiere ninguna configuración extra

**Contras:**

- Bloquea el hilo principal en archivos medianos y grandes
- El spinner de carga no se anima durante el freeze (el `isLoading = true` ya está seteado pero el repintado no ocurre)
- SheetJS queda en el bundle principal, aumentando el tiempo de primera carga para todos los usuarios
- El navegador puede marcar la página como "no responde" en dispositivos lentos con archivos cercanos a 5 000 filas
- Hacia que este componente fuese muy pesado y esa responsabilidad para un solo archivo lo sofocaba y no le permitia ser escalable a nuevos cambios

---

### Opción B — Web Worker con transferencia de ArrayBuffer

Mover el parseo, validación, deduplicación y agrupación a un Web Worker (`carga-guias.worker.ts`). El componente lee el archivo como `ArrayBuffer` con `FileReader` y lo **transfiere** (no lo copia) al worker. El worker responde con el resultado serializado como array de entradas de `Map`.

**Pros:**

- El hilo principal queda completamente libre: la UI permanece fluida y el spinner se anima
- SheetJS solo vive en el worker, fuera del bundle principal de la aplicación. Lo que permitia un menor consumo
- La transferencia de `ArrayBuffer` (`Transferable`) evita duplicar el uso de memoria RAM — el hilo principal cede la propiedad del buffer al worker sin copiarlo
- Los errores del worker (archivo corrupto, validaciones fallidas) se capturan limpiamente y se reportan al componente via `postMessage({ ok: false, error })`
- El worker se instancia y termina (`worker.terminate()`) por cada operación: sin estado residual entre cargas

**Contras:**

- Añade un archivo adicional al módulo (`workers/carga-guias.worker.ts`)
- El `Map<string, any[]>` no es serializable por `postMessage` (structured clone no soporta `Map`) → se debe serializar como `[...grupos.entries()]` y reconstruir en el componente con `new Map(data.pedidosPorTienda)`
- Requiere que Angular CLI esté configurado para compilar Web Workers (soporte nativo desde Angular 8+ con `webWorkerTsConfig`)
- La comunicación asíncrona complica levemente el seguimiento del flujo para nuevos desarrolladores

---

### Opción C — Procesar el archivo en el backend (servidor)

Enviar el archivo `.xlsx` al backend API Gateway para que sea el servidor quien lo parsee, valide y devuelva el JSON de pedidos agrupados.

**Pros:**

- El cliente no necesita SheetJS ni ninguna librería de parseo
- La validación de columnas requeridas y duplicados ocurre en un entorno controlado

**Contras:**

- Aumenta significativamente el payload de la petición HTTP: un Excel de 5 000 filas puede superar los 5–8 MB en Base64
- API Gateway tiene un límite de payload de 10 MB; archivos grandes estarían en el límite
- Añade latencia de red al flujo — el usuario espera el round-trip antes de ver cualquier resultado

---

## Decisión

**Se eligió:** Opción B — Web Worker con transferencia de `ArrayBuffer`

**Razón principal:** Es la única opción que elimina el freeze de UI sin aumentar la carga de la infraestructura backend ni superar los límites de payload de API Gateway. La transferencia de `ArrayBuffer` resuelve el problema de memoria sin overhead de copia, y el confinamiento de SheetJS al worker mejora el tiempo de carga inicial de la aplicación. La complejidad de serialización del `Map` es un costo menor y bien acotado.

---

## Consecuencias

### Positivas

- La UI permanece completamente fluida durante el parseo de archivos grandes (hasta 5 000 filas)
- El spinner de "Procesando archivo..." se anima correctamente en todos los casos
- SheetJS queda fuera del bundle principal: los módulos que no usan carga masiva no pagan el costo de esa dependencia
- Las validaciones del archivo (filas, columnas, duplicados, columnas requeridas) están centralizadas y aisladas en el worker, fáciles de extender sin tocar el componente
- La memoria RAM no se duplica al pasar el archivo: el `ArrayBuffer` se transfiere, no se copia

### Negativas / Compromisos

- El código que antes era un método privado síncrono ahora involucra comunicación asíncrona entre hilos y serialización manual del `Map`
- `convertirExcelAJson()` y `agruparPorTienda()` quedaron como métodos muertos en el componente (código legado no eliminado aún) — generan confusión sobre cuál es el flujo activo
- Un desarrollador nuevo que no conozca la arquitectura de Web Workers puede tener dificultades para seguir el flujo completo de `uploadFile()`
- Si Angular CLI no está correctamente configurado para compilar workers, la funcionalidad falla completamente en nuevos entornos de desarrollo

### Neutrales

- El worker se instancia por cada operación de carga (no hay un worker persistente en background) — comportamiento equivalente en consumo de recursos para el patrón de uso actual (una carga cada varios minutos)
- Las validaciones del worker (`MAX_ROWS`, `MAX_COLUMNS`, `REQUIRED_COLUMNS`) están hardcodeadas en el worker; los campos excluidos de deduplicación (`fieldsToExclude`) se reciben como parámetro desde el componente

---

## Implementación

| Módulo / Archivo | Impacto |
|---|---|
| `carga-guias.component.ts` | Se reemplazó la llamada directa a `convertirExcelAJson()` + `agruparPorTienda()` por `procesarExcelEnWorker(buffer)`. Los métodos originales permanecen como código muerto. Se añadió `leerArchivoComoBuffer()` para obtener el `ArrayBuffer` antes de transferirlo |
| `workers/carga-guias.worker.ts` | Archivo nuevo. Contiene toda la lógica de parseo (SheetJS), validación estructural, conversión a JSON, deduplicación y agrupación por tienda |
| `angular.json` / `tsconfig` | Requiere soporte de Web Workers habilitado en el proyecto Angular (disponible por defecto desde Angular CLI 8+) |

### Flujo antes (Opción A — reemplazado)

```
uploadFile()
  → leerArchivoComoBuffer()        [FileReader, async]
  → XLSX.read(buffer)              [SheetJS, SÍNCRONO — bloquea UI]
  → convertirExcelAJson(rawData)   [loop N×M — bloquea UI]
  → agruparPorTienda(jsonData)     [loop N — bloquea UI]
  → continúa con validaciones...
```

### Flujo después (Opción B — activo)

```
uploadFile()
  → leerArchivoComoBuffer()                       [FileReader, async]
  → procesarExcelEnWorker(buffer)                 [transfiere buffer al worker]
        ↓ hilo principal libre — UI fluida
      [Worker] XLSX.read + convertir + agrupar
        ↓ postMessage({ ok, pedidosPorTienda, totalPedidos })
  → new Map(data.pedidosPorTienda)                [reconstruye el Map]
  → worker.terminate()
  → continúa con validaciones...
```

---

## Estado futuro

Esta decisión se revisará si:

- El volumen de pedidos por carga supera las 5 000 filas de forma frecuente — en ese caso habría que evaluar paginación del procesamiento dentro del propio worker o un streaming approach
- Se requiere procesar múltiples archivos en paralelo — se podría mantener un pool de workers en lugar de instanciar uno por operación
- Angular introduce un mecanismo nativo mejor para off-thread processing (por ejemplo, integración con `TaskWorklet` si llega a los browsers)
- Se detecta que el overhead de serialización del resultado (arrays de miles de objetos) es un cuello de botella medible — en ese caso se evaluaría `SharedArrayBuffer` con sincronización via `Atomics`

---

## Referencias

- Commit `c62fbe3` — "Carga de guias mejora en rendimiento": introduce el worker y reemplaza el flujo síncrono
- [MDN Web Docs — Using Web Workers](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Using_web_workers)
- [MDN — Transferable objects](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Transferable_objects)
- [Angular Docs — Web Workers](https://angular.dev/ecosystem/web-workers)
- [SheetJS — Browser parsing](https://docs.sheetjs.com/docs/demos/local/file/)
- `CARGA-GUIAS.md` — documentación funcional completa del componente consumidor
- `CARGA-GUIAS-WORKER.md` — documentación técnica del worker
