## Autor: Iker Acevedo

Fecha creacion: 2026-08-26

Estado: produccion

## Componente: progreso-lote

**Selector:** `app-progreso-lote`
**Ubicación:** `components/progreso-lote/`

---

## ¿Qué hace?

La pantalla de progreso en vivo mientras el lote de archivo grande está abierto. Solo lee del store — quien orquesta el polling es `ingreso-devoluciones.component.ts`, este componente solo muestra gráficamente el avance.

Muestra: cabecera con ícono/badge de estado, barra de progreso, "Progreso Global" con porcentaje grande, tiempo transcurrido + estimación de lo que falta, y los 3 resultados (Resueltas/Ya procesadas/Rechazadas) en tarjetas separadas — nunca sumados, porque significan cosas distintas para el operario.

---

## Reloj vivo

```typescript
ngOnInit()  setInterval(() => actualizarTranscurrido(), 1000)
```

Recalcula `segundosTranscurridos` cada segundo, a partir de `store.loteActivo().FechaInicio` — para que el reloj se sienta vivo sin depender de que llegue una respuesta nueva del polling (que es cada 2.5s, no cada 1s).

`tiempoEstimadoRestante` usa `estimarSegundosRestantes()` de `rules.ts` — puede devolver `null` si todavía no hay ritmo observable, y el template no muestra nada en ese caso (mejor nada que un número absurdo).

---

## Estados visuales cubiertos

| `Estado` | Qué muestra |
| -------- | ----------- |
| `Pendiente` | Barra animada con franjas en movimiento (el porcentaje real es 0, pero comunica "esto está vivo", no congelado) |
| `EnProceso` | Barra real, reloj, badge naranja |
| `WorkerDetenido: true` | Alerta: el proceso lleva más de 2 minutos sin actividad — **explica que no se perdió nada** y que reintentar es seguro |
| `Fallido` | Alerta de error con `MensajeError` |

**Nota sobre lo que este componente NO muestra**, por decisión explícita durante el diseño: no hay una lista de "actividad reciente" guía-por-guía en vivo, ni un panel de "inventario actualizándose" separado. El backend no expone eventos por guía durante el procesamiento — solo contadores agregados por `GET /estado`; el detalle por guía (`Detalle`) recién llega cuando el job **termina**. Mostrar una lista en vivo habría requerido fabricar datos que el backend no entrega. El inventario, además, se actualiza **en cada lote**, no "al final" — un panel de "en espera" habría sido directamente incorrecto.
