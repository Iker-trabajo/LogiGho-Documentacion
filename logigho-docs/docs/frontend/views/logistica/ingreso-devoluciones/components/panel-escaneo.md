## Autor: Iker Acevedo

Fecha creacion: 2026-08-26

Estado: produccion

## Componente: panel-escaneo

**Selector:** `app-panel-escaneo`
**Ubicación:** `components/panel-escaneo/`

---

## ¿Qué hace?

Pistoleo: lector físico (input + Enter) o cámara en celular (ZXing). Cada guía entra a un [`EscaneoBuffer`](../helpers/escaneo-buffer.md) local; cuando el buffer flushea, este componente dispara su **propio** mini-job por lote (`iniciarLote` + un mini-polling propio) y actualiza el resultado fila por fila.

**A propósito NO usa `IngresoDevolucionesStore.loteActivo`** — ese signal es exclusivo del flujo de archivo grande (`progreso-lote`). Mezclar los dos haría que un escaneo pisara la barra de progreso de un archivo que sigue corriendo en paralelo.

---

## Flujo

```
onKeyPress (Enter) / onScanSuccess (camara)
  procesarEscaneo(valorCrudo)
    guia vacia? -> ignora
    ya vista en esta sesion? -> marca 'duplicada', sonido de error, NO se envia
    si no: marca 'en-cola', buffer.agregar(guia), sonido de exito

buffer flushea (silencio de 3s o tope de 25) -> enviarLote(guias)
  marca 'enviando'
  repo.iniciarLote(guias)
    si falla -> reencolar(guias): vuelve a 'en-cola' y buffer.agregar() de nuevo
               (se reintenta solo en el proximo flush del buffer)
    si ok -> consultarHastaTerminar(jobId, guias)
               mini-polling propio, mismo INTERVALO_POLLING_MS que el flujo grande
               cuando Completado/Fallido -> resolverResultado(jobId, guias)

resolverResultado(jobId, guias)
  Promise.all([ repo.obtenerEstado(jobId, true), repo.obtenerInventarioDeLote(guias) ])
  cruza Detalle (rechazadas) con InventarioDevolucion (resueltas)
  por cada guia del lote de escaneo: marca 'rechazada' o 'resuelta'
```

---

## Limitación documentada explícitamente en el código

**No se distingue `'resuelta'` de `'ya-registrada'` en el pistoleo.** El backend no expone esa granularidad por guía individual dentro de `InventarioDevolucion` — cualquier guía con `Validacion == "OK"` se marca `'resuelta'`, sin importar si fue nueva o si ya estaba procesada de antes. El modelo `EstadoEscaneo` sigue teniendo `'ya-registrada'` definido por si algún día el backend expone ese dato por guía; hoy no se puede pintar con certeza.

Si una guía enviada no aparece ni en `Detalle` (rechazadas) ni en el inventario tras completar el job — caso que en teoría no debería pasar — se marca `'ya-registrada'` como valor de seguridad y se logea con `console.warn`, para no dejar la fila colgada en `'enviando'` para siempre.

---

## Reintento automático en errores de red

Si `repo.iniciarLote(guias)` falla (error de red, no un rechazo de negocio), las guías se reencolan automáticamente: vuelven a `'en-cola'` y se reingresan al buffer. El operario no ve un error explícito por esta falla — simplemente ve la fila un poco más de tiempo en `'en-cola'`, y el buffer las reintenta en su próximo flush.

---

## Sonido: por qué se cambió el del legacy

El módulo legacy (`devolucion-inventario.component.ts`) usaba un sonido de error de **5 segundos** (`playTenebrousSound`) para una guía duplicada. En un flujo de pistoleo rápido con este buffer, un sonido de 5 segundos por cada duplicada se acumularía y bloquearía el ritmo del operario si escanea varias seguidas. Se reemplazó por un beep corto (0.2s): agudo (880Hz, onda seno) para éxito, grave (220Hz, onda cuadrada) para duplicada/error.

---

## Cámara en celular

`detectarDispositivo()` chequea `navigator.userAgent` — en mobile muestra el botón de activar cámara (`ZXingScannerModule`, formatos `CODE_128`/`EAN_13`/`UPC_A`) en vez del input de teclado, porque en ese contexto no hay lector físico de código de barras conectado.
