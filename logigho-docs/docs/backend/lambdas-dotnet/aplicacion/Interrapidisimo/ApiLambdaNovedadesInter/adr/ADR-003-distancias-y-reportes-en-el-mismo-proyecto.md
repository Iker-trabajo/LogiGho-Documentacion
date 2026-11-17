---

## Autor: Iker Acevedo  
Fecha creacion: 2026-11-06  
Estado: aceptada

# ADR-003 — Distancias como rama del Parallel, Reportes con Scheduler y Slack, un proyecto con tres funciones

**Autor:** Iker Acevedo  
**Fecha:** 2026-11-06  
**Estado:** Aceptada (decisión de Iker 2026-11-05; PreProd validada 2026-11-06; producción pendiente)

---

## Contexto

La historia 027 pide dos cosas nuevas sobre las novedades de Inter: calcular a cuántos metros por calle de la dirección del pedido marcó el
mensajero cada novedad, y entregar a la gerencia dos Excel por Slack. Restricciones comprobadas:

- Los callbacks de ws-inter llegan y terminan dentro de la misma hora del disparo (observado en CloudWatch prod 2026-11-05: 200-360 invocaciones en la
  primera media hora). Una rama que corra al inicio de la Step encuentra completos los callbacks del ciclo anterior.
- Un fallo de Google o de Slack **no** puede frenar las liquidaciones ni las demás ramas de `Step_Procesos_Logigho`.
- Un webhook de Slack no puede subir archivos; hace falta un bot.
- El comando de Slack debe responder en menos de 3 s, pero armar los Excel tarda más.
- Pocas lambdas: revisar las existentes antes de crear otra (Iker).

## Opciones consideradas

### Decisión 1 — Dónde corre Distancias

**A. Dentro de `ApiLambdaNovedadesInter`, después de solicitar.** Pros: una sola invocación. Contras: alarga y acopla el ciclo de solicitud; un fallo de Google afectaría a ws-inter.  
**B. Rama nueva del `Parallel` de la Step.** Pros: aislada con su propio `Catch`; no espera a nadie; encuentra los callbacks del ciclo anterior. Contras: retraso máximo de 3 h; la Step de producción se edita con un script.

### Decisión 2 — Cómo se dispara Reportes

**A. Solo programado.** Contras: no sirve para pedirlo entre martes y viernes.  
**B. Scheduler (martes y viernes 8:00 Bogotá) y comando de Slack `/reporte-novedades`, con la misma función.** Pros: un solo código para ambos; el comando responde rápido y se autoinvoca de forma asíncrona. Contras: ruta pública sin authorizer que hay que proteger.

### Decisión 3 — Cuántos proyectos y funciones

**A. Proyectos separados por función.** Contras: duplica pipelines, VPC, SNS, etiquetas y zip.  
**B. Un proyecto, tres funciones** (patrón del dominio inventario): `ApiLambdaNovedadesInter`, `ApiLambdaNovedadesInterDistancias` y `ApiLambdaNovedadesInterReportes`, un solo zip con tres handlers y un CD con lista de funciones.

### Decisión 4 — Canal de entrega

**A. Correo (SES).** Descartado por Iker: Slack es el canal (spec 012, «No entra»).  
**B. Slack con bot** (`files.getUploadURLExternal`, subida de los bytes y `files.completeUploadExternal`).

## Decisión

**1-B, 2-B, 3-B, 4-B.**

**Razón:** la rama del `Parallel` aísla Google del resto de la Step; Reportes comparte código entre el Scheduler y el comando; un proyecto con tres
funciones reduce pipelines y comparte VPC, subred, SG y alarmas, sin cerrar la puerta a separarlas más adelante; Slack es el canal elegido y el bot
sí sube archivos. La seguridad de la ruta pública descansa en la firma HMAC de Slack (ventana de 300 s, tiempo constante), la lista de
usuarios permitidos, el tope de 4 KB y el WAF del REST.

## Consecuencias

**Positivas:** un fallo de Distancias no frena nada (CA6); el comando responde sin esperar los Excel; los costos de Google se acotan con caché y tope por corrida;
un solo pipeline despliega las tres funciones.  
**Negativas:** retraso de hasta 3 h en las distancias; Reportes lee casi todos los seguimientos en cada corrida (M-257); la Step de producción se
modifica con un script (`agregar_distancias_a_step_function.py`) porque no está en Terraform; los secretos de Google y Slack viajan por el variable group.

## Impacto en el código

| Módulo / Repo | Cambio |
| ------------- | ------ |
| `Lambdas_NET_LogiGho` | `ApiLambdaNovedadesInter`: `DistanciasHandler`, `ReportesHandler`, casos de uso y servicios (Google, Slack, Excel); `pipelines/novedades` con lista de funciones; script de la Step de prod |
| `Terraform_Infraestructura_Logigho` | `modules/novedades-inter` (2 funciones, roles, logs, alarmas, Scheduler), `modules/novedades-reportes-slack` (ruta), rama nueva en la Step de PreProd, rol de despliegue con las 3 funciones |
| `Angular SitioLogiGho` | Vista de novedades: columna, filtro, pestaña «Comparar con dirección», Excel |
| `LogighoDB` | Campos nuevos en `SeguimientoNovedadesInter`; colecciones `ReportesNovedadesInter` y `CandadosNovedadesInter` |

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-11-06 | Iker Acevedo | Creación |

## Referencias

- [Distancias](../ApiLambdaNovedadesInterDistancias.md) · [Reportes](../ApiLambdaNovedadesInterReportes.md) · [ADR-001](ADR-001-una-lambda-paso-en-la-step-y-dos-colecciones.md)
- Spec 012 y decisiones 2026-11-05 del workspace (`specs/012_novedades_distancia_reportes/`, `docs/architecture/decisiones.md`).
