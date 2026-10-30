# Métricas — % de entrega por ciudad y transportadora

**Autor:** Iker Acevedo · **Fecha de creación:** 2026-10-29 · **Actualizado:** 2026-10-30 · **Estado:** backend en **producción** (primera corrida verificada); Scheduler de prod apagado a la espera de activarlo

El dominio **`metricas`** agrupa los procesos que **calculan indicadores a partir de la base de datos** y los dejan listos para consultar en el sitio. Su primera métrica es **`ApiLambdaMetricasEntregaCiudad`** (.NET 10): el **porcentaje de entrega de cada transportadora por ciudad**, para saber en qué ciudades le va mejor y peor a cada una y elegir con datos con quién despachar.

!!! info "Cómo leer el estado de cada afirmación"
    - **Confirmado** = está en código, en Terraform o en una verificación hecha (tests, CD en verde, corrida revisada en la base).
    - **Pendiente** = decidido o propuesto, pero no hecho o no verificado. Siempre se dice como tal.

## Mapa de esta sección

| Página | Qué responde |
|---|---|
| **Esta página** | Qué problema resuelve, cómo funciona de punta a punta, glosario, estado |
| [Arquitectura](arquitectura.md) | Las dos puertas de la función, el flujo de una corrida, colecciones y variables |
| [Reglas de clasificación](reglas-clasificacion.md) | Cómo se clasifica cada guía, el %, el mínimo de muestra, normalización |
| [Seguridad](seguridad.md) | Verificación de la firma del JWT, roles, por qué el gateway va sin authorizer |
| [Ajuste visual del porcentaje](ajuste-visual.md) | Regla de gerencia, parámetros y dónde se aplica |
| [API de métricas](../../api/metricas.md) | Contratos de las 6 rutas, errores, CORS |
| [Operación y despliegue](operacion.md) | Terraform, pipelines, variables, stage, corrida manual, reversión |
| [Liberación de oct 2026](../../changelog/metricas-2026-10.md) | Qué salió, qué falló en el camino y cómo se corrigió |

## El problema

`PedidosInter` en producción tiene **~926 000 guías** y crece 50-100 mil por mes. Agrupar toda la colección contra la DocumentDB `t4g.medium` tarda 30-40 s, y API Gateway corta a los 29 s: **no se puede calcular en vivo desde el front** sin cargar la base donde se crean las guías.

Además, los datos de origen vienen sucios: la transportadora tiene variantes y errores de tipeo (`INTER`, `ENIVA`, `INTERRAPIDISO`), la ciudad es texto libre sin código DANE, y los estados son muchos y no homologados.

## La solución

1. **Proceso programado, no consulta en vivo.** Una corrida (domingo 8:00 Bogotá, o manual) lee `PedidosInter` **una sola vez**, clasifica en memoria y escribe un **resumen** por mes × ciudad × transportadora. El front solo lee ese resumen (milisegundos).
2. **Clasificación configurable** sin desplegar código: las listas de estados de entrega/devolución, el mapa de transportadoras, las excluidas y los alias de ciudad viven en una colección editable desde el panel de administración.
3. **Normalización al leer:** la transportadora se lleva a un código canónico (los typos se mapean), la ciudad se resuelve al DANE `mun8` del catálogo `Ciudades`, el estado se compara normalizado (mayúsculas, sin tildes).
4. **Auditoría por corrida:** guías leídas, clasificadas por motivo, % de ciudad resuelta, estados observados con su clase y ciudades sin reconocer.
5. **Una sola función con dos puertas:** un evento `{"tipo":"CORRIDA"}` (Scheduler o autoinvocación) ejecuta el cálculo; un evento de API Gateway enruta a las lecturas y a la administración.
6. **Siempre el dato real en la base.** La base, las corridas y el backend guardan y calculan el porcentaje real. El **ajuste visual** (spec 005) solo cambia lo que el front muestra y se explica en su [propia página](ajuste-visual.md).

## Glosario

| Término | Significado |
|---|---|
| **Corrida** | Una ejecución del cálculo. Tiene origen (`PROGRAMADA` o `MANUAL`), alcance y estado (`EN_CURSO`, `FINALIZADA`, `FALLIDA`, `OMITIDA`). Una `EN_CURSO` de más de 20 min se considera caída |
| **Alcance** | Qué meses recalcula: `TODO`, `ULTIMOS_6` o `ULTIMOS_3`. Los meses fuera del alcance conservan su último cálculo |
| **Guía final** | Guía clasificada como ENTREGA o DEVOLUCION. El % y el mínimo de muestra se calculan solo sobre finales |
| **% de entrega** | `entregas ÷ (entregas + devoluciones)`. Las NO_CUENTA (tránsito, anuladas, excluidas) no entran |
| **Mínimo de muestra** | Guías finales mínimas (por defecto 30) para que una celda cuente en "mejor opción" y en los rankings. Menos que eso se marca "sin muestra" |
| **NO_CUENTA** | Guía que no entra al cálculo, con su motivo: tránsito, transportadora excluida (XCARGO) o desconocida, sin ciudad, sin fecha de carga |
| **Candado** | Documento atómico en Mongo que impide dos corridas a la vez (RN-08) |
| **mun8** | Código DANE de 8 dígitos del municipio, clave para unir una ciudad de texto libre con el catálogo `Ciudades` |
| **Ajuste visual** | Parámetros (`activo`, `porcentajeMinimo`, `modo`) con los que el front eleva los porcentajes que muestra, sin tocar los datos |

## Estado a 2026-10-30

!!! success "Confirmado"
    - **PreProd:** backend desplegado por pipeline, rutas en el stage `PreProd` del REST `8u5isw5u30`.
    - **Producción:** `terraform apply` de `environments/prod/metricas` (45 recursos), función `Active` dentro de la VPC de prod, CD de prod ejecutado, stage `Produccion` de `7elgyten88` publicado, `GET` sin token responde 401 `SIN_TOKEN`.
    - **Primera corrida `TODO` en prod** (2026-10-30): 926 476 guías leídas en **1 min 53 s**, **98,1 %** con ciudad resuelta, 25 023 documentos escritos, 1 118 ciudades. Los totales se contrastaron con conteos directos sobre `PedidosInter` (ver [Liberación](../../changelog/metricas-2026-10.md)).
    - **Tests:** 201 en la lambda y 93 en las dos vistas del front, en verde.
    - **Seguridad:** la lambda verifica la firma del JWT de Cognito antes de confiar en el `sub` ([Seguridad](seguridad.md)).
    - **Datos de prod:** configuración inicial, índices y los módulos de menú 169 y 170 insertados.

!!! warning "Pendiente"
    - **Scheduler de prod apagado** (`estado_programacion = "DISABLED"`). Se activa después de validar la vista con gerencia.
    - **Ajuste visual apagado en prod** hasta que gerencia revise la vista previa del panel.
    - **Front en producción:** despliegue de Angular (`preprod` → `main`) por confirmar.
    - Estados `Facturado` (Inter), `INDEMNIZADA` y `EN PROCESO DE INDEMNIZACION` (Envía) siguen como NO_CUENTA hasta que Iker decida.
    - Mejoras abiertas en `backlog.md` (M-130/M-131: un solo gateway y deployment del stage; M-124: índice en `FechaCarga`).
