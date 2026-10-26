---
autor: Iker Acevedo
fecha_creacion: 2026-10-22
estado: aceptada
---

# ADR-001 — Hosting: ASP.NET Core en Lambda detrás de ApiIntegracionesExternas

**Fecha:** 2026-10-22
**Estado:** Aceptada

---

## Contexto

- Volumen: ~50k pedidos/mes (ago 54.250, sep 49.876). Meta 10k/día al lanzar la API y luego 50k/día.
- `ApiLambdaCrearGuia`: ~100 invocaciones/día, promedio 4 s, máximo 123 s. `ApiLambdaCargarGuias` llega al timeout de 900 s.
- Las transportadoras responden lento y con variaciones.
- Cuenta prod: límite 1000 Lambdas concurrentes, pico observado 134, cero throttles.
- DocumentDB prod: una instancia `db.t4g.medium`, CPU máxima 99%, 495 conexiones pico. Es el cuello de botella real.
- Costo AWS actual: USD 680–1040/mes; Lambda ~USD 100/mes.
- `ClusterLogigho` (ECS) existe sin servicios: allí corrían las APIs de Hoko, ya apagadas. No existe ALB/NLB ni VPC link para LogiGho.

## Opciones

### A — ASP.NET Core en Lambda + worker SQS (elegida)

- **Pros:** costo incremental ~USD 20–50/mes; mismo stack que el equipo opera; escala solo; el límite de 15 min no aplica porque cada mensaje es una guía; el de 29 s no aplica porque el API encola y responde `202` si tarda; concurrencia reservada pone techo a las conexiones de DocumentDB; el mismo código corre en ECS sin reescribir.
- **Contras:** cold start (mitigable); un pool de conexiones por contenedor.

### B — ECS Fargate

- **Pros:** sin cold start, conexiones calientes, procesos largos.
- **Contras:** ~USD 250–300/mes extra (2+ tareas, NLB, VPC link); infraestructura nueva que hoy no existe; no resuelve DocumentDB.

## Decisión

Opción A. El código va en `ServicioPublicoLogigho/Guias/` del repo de lambdas, según la regla del equipo.

**Cuándo reconsiderar ECS:** concurrencia sostenida alta que dispare costos de Lambda, necesidad de procesos largos o problemas de cold start que afecten el SLA.

## Consecuencias

- `Api` usa `Amazon.Lambda.AspNetCoreServer.Hosting`; el worker usa un consumidor de mensajes intercambiable.
- Lambdas en ambas subnets privadas y con concurrencia reservada.
- PreProd necesita subir su límite de 10 Lambdas concurrentes antes de pruebas de carga.
