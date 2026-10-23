---

## Autor: Iker Acevedo  
Fecha creacion: 2026-10-18  
Estado: aceptada

# ADR-001 — Una lambda, un paso nuevo en la Step y dos colecciones

**Autor:** Iker Acevedo  
**Fecha:** 2026-10-18  
**Estado:** Aceptada

---

## Contexto

Hay que saber **por qué** una guía de Interrapidísimo queda en novedad (motivo y ubicación). La API oficial de Inter solo da
estado; un proveedor externo (ws-inter) lo ofrece, una guía por solicitud, con respuesta asíncrona por callback.

Restricciones comprobadas en el código y en producción:

- `ApiLambdaProcesoAutomaticoInter` no tiene pipeline ni pruebas, se despliega a mano, construye sus dependencias dentro
  (no es testeable) y reescribe cada pedido completo (`ReplaceOne`) en cada corrida.
- Su detección de novedad está rota: usa los hijos de `PadreId == 28` del catálogo (solo "Custodia"); apenas 8 de ~900 000 pedidos
  tienen `Estado Novedad`.
- `PedidosInter` mezcla transportadoras y tiene más de 100 campos planos.

## Opciones consideradas

### Decisión 1 — Dónde se dispara

**A. Dentro de `ApiLambdaProcesoAutomaticoInter`** (invocar la nueva al final).  
Pros: una sola corrida. Contras: toca código legacy sin pruebas ni pipeline; lee el estado de la corrida anterior; si ws-inter falla afecta la sincronización de estados.

**B. Paso nuevo en `Step_Procesos_Logigho`, después de la lambda Inter.**  
Pros: lee los estados ya actualizados; fallos aislados (`Catch`); legacy intacto. Contras: una invocación más en la Step.

### Decisión 2 — Una o dos funciones Lambda

**A. Dos** (solicitar y webhook). Más aislamiento de permisos/timeouts; doble de proyectos, pipelines y despliegues.  
**B. Una con dos rutas.** Ambas usan Mongo, la misma colección y el mismo dominio; volumen bajo (~900 callbacks por corrida). En microservicios la unidad es el contexto acotado, no la función; se separa cuando cambian mucho los permisos, la escala o los timeouts.

### Decisión 3 — Dónde guardar el resultado

**A. Subdocumento o campos planos en `PedidosInter`.** Lo mantiene todo junto, pero engorda una colección compartida que además la legacy reescribe.  
**B. Colección `SeguimientoNovedadesInter` + un solo campo `MotivoNovedadInter` en `PedidosInter`.**

## Decisión

**1-B, 2-B, 3-B.**

**Razón:** el paso en la Step no toca la legacy y es el único que ve los estados ya actualizados; una función reduce proyectos,
pipelines y cosas que monitorear sin cerrar la puerta a separarla (el código ya tiene dos casos de uso independientes); y la
colección propia guarda historial, geo y trazabilidad de la sincronización sin ensuciar `PedidosInter`, donde basta un texto
legible (`MotivoNovedadInter`).

## Consecuencias

**Positivas:** legacy sin cambios; fallos de ws-inter no afectan estados; el detalle se consulta por número de guía; no se
consulta dos veces lo mismo (el seguimiento es la memoria).  
**Negativas:** la lambda legacy puede pisar `MotivoNovedadInter` al reescribir el pedido (se mitiga reponiéndolo en cada
corrida); dos colecciones que mantener consistentes; la Step de producción se modifica con un script (no está en Terraform).

## Impacto en el código

| Módulo / Repo | Cambio |
| ------------- | ------ |
| `Lambdas_NET_LogiGho` | `Interrapidisimo/ApiLambdaNovedadesInter` (src + test), `pipelines/novedades` |
| `Terraform_Infraestructura_Logigho` | `modules/novedades-inter`, `environments/{preprod,prod}/novedades`, Step de PreProd |
| `LogighoDB` | Colección `SeguimientoNovedadesInter` (+3 índices) y campo `MotivoNovedadInter` en `PedidosInter` |

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-10-18 | Iker Acevedo | Creación |

## Referencias

- [ApiLambdaNovedadesInter](../ApiLambdaNovedadesInter.md) · [Modelos](../dominio/modelos.md)
