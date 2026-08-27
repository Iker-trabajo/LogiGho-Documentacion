## Autor:
Fecha creacion: 2026-08-26
Estado: aceptada

# ADR-001 — Tres funciones Lambda en un solo proyecto, orquestadas por invocación asíncrona

**Autor:** Iker Acevedo
**Fecha:** 2026-08-26
**Estado:** Aceptada

---

## Contexto

El módulo legacy de devoluciones masivas causaba picos de 99% de CPU en DocumentDB al procesar archivos grandes con lógica dispersa desde el navegador, sin control de lote ni límite de conexiones. Había que rediseñarlo desde cero, y dos decisiones de arquitectura quedaban abiertas desde el principio:

1. ¿Cuántos handlers/funciones Lambda, y cómo se comunican entre sí?
2. ¿El procesamiento pesado corre síncrono detrás de una API, o asíncrono en segundo plano?

Restricción dura: API Gateway corta cualquier integración a los **30 segundos**. Procesar hasta 3.000 guías contra Mongo y contra 2 APIs externas de transportadoras puede tardar varios minutos.

---

## Opciones consideradas

### Opción A — Todo en una sola función HTTP síncrona

Un único handler recibe el archivo y procesa todo antes de responder.

**Pros:** más simple de entender, un solo despliegue.
**Contras:** inviable con el techo de 30s del gateway para cualquier archivo de más de unas pocas decenas de guías. El operario tendría que mantener la conexión HTTP abierta minutos enteros desde el navegador — no es viable ni siquiera técnicamente (los navegadores y los balanceadores intermedios cortan conexiones largas).

### Opción B — Microservicios separados (3 proyectos .NET, 3 repos o 3 paquetes NuGet internos)

Cada función en su propio proyecto/repositorio, comunicándose por colas (SQS) o eventos.

**Pros:** aislamiento total, cada equipo podría evolucionar su parte sin acoplarse.
**Contras:** las 3 funciones comparten el mismo dominio (`JobDevolucion`, `GuiaProcesada`, enums), los mismos repositorios (`JobRepository`, `DevolucionRepository`) y las mismas reglas de negocio. Separarlas en proyectos distintos obligaría a duplicar esas clases o publicarlas como paquete NuGet interno — mantenimiento extra sin beneficio real para un módulo de este tamaño, operado por un equipo pequeño.

### Opción C — Un proyecto, tres funciones (`function-handler` distintos), orquestadas por invocación Lambda-a-Lambda asíncrona

Un mismo código base con 3 `aws-lambda-*.json` (uno por función), cada uno con su propia memoria/timeout/rol. `IniciarHandler` crea el job y responde rápido; dispara a `WorkerHandler` con `InvocationType.Event` (fire-and-forget, sin pasar por API Gateway, sin el techo de 30s); `EstadoHandler` es una consulta de solo lectura, barata, para polling.

**Pros:** cero duplicación de dominio/infraestructura (todo vive en el mismo proyecto). Cada función igual se despliega, escala y factura por separado — no es un monolito en ejecución. El Worker puede correr hasta 15 minutos, el máximo de Lambda, sin ningún límite de API Gateway de por medio.
**Contras:** un solo repositorio de código para 3 funciones exige disciplina para no acoplarlas por accidente (mitigado con interfaces claras: `IJobRepository`, `IDevolucionRepository`, etc.).

---

## Decisión

**Se eligió:** Opción C.

**Razón:** resuelve la restricción de los 30 segundos sin pagar el costo de mantenimiento de microservicios reales, para un módulo cuyo dominio es compartido y pequeño. El patrón "iniciador rápido + worker asíncrono + consulta de estado por polling" es el estándar de facto para trabajo pesado detrás de una API Gateway con Lambda.

---

## Consecuencias

**Positivas:** el operario obtiene respuesta en ~200ms al subir un archivo de 3.000 guías, en vez de esperar minutos o recibir un timeout del gateway. El Worker puede reintentarse a sí mismo (ver [ApiLambdaDevolucionesMasivo.md](../ApiLambdaDevolucionesMasivo.md#qué-pasa-cuando-el-worker-muere-y-se-reactiva--cambia-el-jobid)) sin que el operario tenga que hacer nada.

**Negativas:** el front nunca sabe en tiempo real (push) cuándo termina un job — tiene que hacer polling cada 2.5s. Es una limitación aceptada: no hay WebSockets ni Server-Sent Events en la arquitectura de este módulo, y el costo de agregarlos no se justificaba para una consulta tan barata como `findOne` por `_id`.

El Worker no tiene ruta propia en API Gateway — es una decisión de seguridad, no solo de arquitectura: si la tuviera, cualquiera con el nombre de la función podría dispararlo saltándose toda la validación de usuario que hace `IniciarHandler`.

---

## Impacto en el código

| Componente | Rol |
| ---------- | --- |
| `IniciarHandler` | Crea el job, dispara al Worker, responde rápido |
| `WorkerHandler` | Sin ruta HTTP. Procesa por lotes, se reinvoca a sí mismo si hace falta |
| `EstadoHandler` | Consulta de solo lectura para el polling del front |
| `aws-lambda-iniciar.json`, `aws-lambda-worker.json`, `aws-lambda-estado.json` | Un `function-handler` distinto cada uno, memoria/timeout propios |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-08-26 | Iker Acevedo | Decisión inicial y documento. |

---

## Referencias

- [ApiLambdaDevolucionesMasivo.md](../ApiLambdaDevolucionesMasivo.md) — vista general del módulo y las 3 funciones.
