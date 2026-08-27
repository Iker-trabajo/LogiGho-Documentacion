## Autor: Iker Acevedo

Fecha creacion: 2026-08-26

Estado: produccion

## Helper: EscaneoBuffer

**Ubicación:** `helpers/escaneo-buffer.ts`
**Tests:** `escaneo-buffer.spec.ts` (con `jasmine.clock()`)
**Lo usa:** [`panel-escaneo`](../components/panel-escaneo.md)

---

## ¿Qué hace?

Acumula guías escaneadas y decide cuándo enviarlas en lote: al silencio (`esperarMs` sin un nuevo escaneo) o al llegar al tope (`topeGuias`), lo que pase primero. No sabe nada de HTTP ni de Angular — solo agrupa y dispara `onEnviar` en el momento correcto. Es una clase plana, no un componente ni un servicio inyectable, para poder testearla con `jasmine.clock()` sin mockear nada de red.

```typescript
new EscaneoBuffer({ esperarMs, topeGuias, onEnviar: (guias) => ... })

.agregar(guia)      // encola y reprograma el temporizador; si llega al tope, envia YA
.enviarAhora()      // fuerza el envio inmediato de lo que haya en cola (boton "Procesar ahora")
.pendientes         // cuantas guias hay en cola sin enviar
.destruir()         // cancela el temporizador pendiente, sin enviar nada — llamar en ngOnDestroy
```

---

## Por qué existe

Sin este buffer, cada guía pistoleada dispararía su propia llamada `POST /iniciar` — un job de 1 guía por escaneo. Con un operario escaneando cada 2-4 segundos, eso serían decenas de jobs pequeños por minuto, cada uno con su propio `Worker` invocado, su propio `findOne` de polling — exactamente el tipo de carga sin control contra el backend que este módulo entero fue construido para evitar. Agrupar los escaneos en lotes de hasta 25 (o al primer silencio de 3 segundos) reduce eso a un puñado de jobs por sesión de pistoleo.

---

## Observaciones

- `destruir()` es distinto de `enviarAhora()`: al destruir el componente (el operario navega a otra pantalla), lo correcto es **cancelar** cualquier envío pendiente, no forzarlo — el componente ya no va a estar para procesar la respuesta.
- El buffer no reintenta nada por su cuenta — si `onEnviar` falla, es responsabilidad de quien lo usa (`panel-escaneo`) decidir qué hacer con esas guías (ver [`panel-escaneo`](../components/panel-escaneo.md) — las reencola llamando a `agregar()` de nuevo).
