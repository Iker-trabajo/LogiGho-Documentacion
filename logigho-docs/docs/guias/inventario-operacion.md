# Inventario — operación, go-live y rollback

**Autor:** Iker Acevedo · **Fecha de creación:** 2026-10-21 · **Estado:** ensayado completo en PreProd el 2026-10-10 (37 pasos en verde, rollback incluido); producción pendiente

Resumen operativo del sistema [`ApiLambdaInventario`](../backend/inventario/index.md). El runbook completo, con comandos exactos y criterios de GO/NO-GO, vive en el espacio de trabajo del dominio (`LogiGho-Dominio-Inventario/08-golive/RUNBOOK-GOLIVE.md`); esta página resume el orden, los criterios y el rollback para que el equipo lo entienda sin ese archivo.

!!! danger "Reglas de oro"
    1. Si un criterio de **GO** no se cumple, **no se avanza**: se corrige o se hace el rollback de ese paso.
    2. **Toda escritura en datos de producción la ejecuta Iker, con respaldo.** Los scripts simulan por defecto, exigen `--aplicar` y piden una frase exacta de confirmación.
    3. Durante la ventana **no se registran ingresos, ajustes ni anulaciones** de guías: esas escrituras manuales no se recuperan solas (las de pedidos sí).
    4. Los respaldos contienen datos de producción: carpeta local, **no se suben a git ni a ninguna nube**.

## Estado a la fecha

| Pieza | Estado |
|---|---|
| Backend `ApiLambdaInventario` | Confirmado: en el repo de Azure, rama `release/2026-10-19-inventario`; ~1.370 pruebas unitarias en verde |
| Infraestructura PreProd (Terraform) | Confirmado: roles mínimos, alarmas, dashboard; `plan` = «No changes» |
| Despliegue PreProd por Azure DevOps (`Lambdas-Inventario-CD-PreProd`) | Confirmado: verificado en AWS |
| VAL-1, VAL-2, VAL-3, ensayo del go-live, E2E | Confirmado: sin fallos del inventario |
| Índices en producción | Confirmado: `ix_idproducto_guia` creados y verificados (`IXONLYSCAN`) y `ux_clave` existe |
| Código y funciones en producción | **Pendiente**: nada desplegado; se crea con Terraform antes de la ventana |
| Limpieza de datos, repunte de rutas | **Pendiente**: ocurren en la ventana |

## Roles y accesos que hacen falta

| Qué | Para qué |
|---|---|
| Usuario DocumentDB con lectura y escritura en `LogighoDB`, creación de índices y de la colección temporal `zz_prueba_borrar` (el de solo lectura **no** sirve) | Limpieza, índices, activación |
| Túnel SSH a DocumentDB y el certificado de la base | Todos los scripts |
| AWS: `lambda:Put/Get/DeleteFunctionConcurrency` sobre las dos gemelas | Pausa y rollback |
| AWS: `lambda:InvokeFunction` sobre las funciones nuevas | Barrido, recálculo, humo |
| AWS: `rds:CreateDBClusterSnapshot` (DocumentDB usa la API de RDS) | Snapshot |
| `terraform apply` de `environments/prod/inventario` + service connection `aws-prod-inventario`, variable group `inventario-prod-secretos` y Environment `produccion` (con aprobadores) en Azure DevOps | Infraestructura y CD |
| `apigateway:PutIntegration`, `CreateDeployment`, `UpdateStage` sobre `7elgyten88` y `lambda:AddPermission` | Repunte |

## Línea de tiempo

### Antes de la ventana (D-7 … D-1) — sin cambiar el comportamiento actual

1. **Código y decisiones** (D-7). Código mergeado a `main` por el flujo `feature/*` → `preprod` → `main`; VAL-3 aprobado con `FILTRO_ACTIVACION=on`. Terraform y el CD de producción aplicados **antes** de la ventana: crean las 5 funciones (las de escritura **sin rutas**) y los recursos de observabilidad. Variables: `FORMULA_CONSUMO=v2`, `FILTRO_ACTIVACION=on`, `ZONA_HORARIA_PEDIDOS=America/Bogota`, `CORS_ORIGENES=*`. Regla del barrido creada **deshabilitada** (decisión D1: pasarla a habilitada en simulación).
2. **Prueba de DocumentDB** (D-7): `indices_golive.py --probar-sparse` crea y borra una colección temporal. GO: «este servidor SI soporta unico+sparse» (INV-029). NO-GO: no se crea el índice único.
3. **Ensayo de solo lectura en producción** (D-3): comprobar que el alcance son los **65 productos activos de BTK** (la unión `Productos.IdTienda ↔ Tienda.Id` + `Ecosistema=BTK` debe reproducir la lista de la auditoría), simular la limpieza (anotar la **huella del plan**, el número de acciones y cuánto tarda) y tomar la línea base de invariantes (se espera suciedad).
4. **Preparación sin riesgo** (D-3…D-1): índices **previos** no únicos (`ix_idproducto_guia`, `ix_guia`, `ix_coleccion_iddocumento`) y **campos de activación** (`ManejaInventario`, `FechaInicioInventario`, `Tienda.ServicioInventario`). Las lambdas actuales los ignoran. Productos sin ingreso registrado **no se activan**.
5. **Avisos:** al Jefe de Logística y a quien liquida (nada de liquidaciones manuales durante la ventana).

### La ventana (fin de semana, reservar 8–10 h; se estima 4,5 h)

Se marca T+0:00 al inicio. Corte de seguridad: **si a T+7:30 no se llegó al GO final, rollback y se reprograma**; no se improvisa de noche.

| T+ | Paso | Qué hace | Criterio GO |
|---|---|---|---|
| 0:00 | **P0** Preparación | Túnel abierto, conexión probada, perfil AWS de prod verificado | Todo marcado |
| 0:15 | **P1** Pausar escrituras | `reserved-concurrent-executions 0` en las dos gemelas (apagado instantáneo y reversible); esperar 10 min | `Invocations = 0` en los últimos 10 min; el script de limpieza además se niega a escribir si el último movimiento tiene menos de 10 min |
| 0:30 | **P2** Snapshot + línea base | Snapshot del clúster DocumentDB y `verificar_invariantes --fase pre` | Snapshot `creating`/`available`; línea base guardada |
| 0:45 | **P3** Simular la limpieza | Misma huella que en D-3, o explicar por qué cambió | Sin `PLAN INVALIDO`; solo acciones sobre los 65 productos |
| 1:15 | **P4** Decisión G1 | Humano: P1 verificado, huella explicada, revisión manual aceptada, disco y túnel estables | 5 casillas |
| 1:20 | **P5** Aplicar la limpieza | Respaldo (JSON canónico + sha256) → historial → borrar (R1/R3/R4/R5) → sellar (`slot`+`clave`) → insertar (R2/R7) → replanificar y exigir 0 acciones | `OK: 0 acciones pendientes`, sin conflictos |
| 3:00 | **P6** Índices únicos | `ux_clave` único + sparse en las 5 colecciones | 5 `[CREADO]`, verificación `[OK]` |
| — | **P7** Infra y código | Se hace antes; en la ventana solo se verifica: `terraform plan` sin cambios, 5 funciones `Active`, regla del barrido `DISABLED`, trazabilidad responde 401 sin token | Plan sin cambios |
| 3:10 | **P8** Repunte | Ver [abajo](#p8-repuntar-las-dos-rutas). **Aquí se reabren las escrituras** | Humo en verde |
| 3:40 | **P9** Barrido en simulación | Sobre los 65 productos | `Insertar + Corregir + Borrar ≈ 0`; lo que no sea 0 son revisiones manuales aceptadas |
| 3:50 | **P10** Recalcular el resumen | `IdsRecalcular` con los 65 ids | `Recalculados = 65`, `Completo = true` |
| 4:00 | **P11** Invariantes POST (G2) | `verificar_invariantes --fase post --aceptadas …` | `CUMPLE (0 bloqueantes)` |
| 4:15 | **P12** Menú y front | Registrar el módulo de trazabilidad y desplegar el front con la pantalla habilitada | Un Jefe Logistica ve la pantalla y recibe 200; un usuario sin rol recibe 403 |
| 4:35 | **P13** Cierre | Conservar respaldos ≥ 30 días, reactivar la liquidación si se deshabilitó, observar 24 h | Sin `ERROR` en los logs |

**Tiempos.** El alcance real BTK es de **~607 mil acciones** (592.038 sellos, 10.027 R1, 2.113 R3, 401 R4, 190 R5, 1.894 R2, 90 R7). Con los costos medidos en Atlas, P5 dura ~90 min; con margen ×2 por DocumentDB, hasta ~3 h, más 20–40 min de lectura y verificación. La simulación de D-3 da la cifra real.

**Invariantes de P11** (por producto activo): **I1** cada guía×producto cumple `S−A−E−D−P = 0` o la cantidad del pedido · **I2** cero claves duplicadas, nulas o documentos sin sellar (salvo aceptados) · **I3** tránsito ≥ 0 · **I4** el resumen guardado = el recálculo. Las advertencias (`SIN_PEDIDO`, `PEDIDO_REPETIDO`, `CIERRE_FALTANTE`, `ACTUAL_NEGATIVO`) se leen pero no bloquean.

**Revisión manual.** Los casos que la limpieza no toca solos se aceptan por escrito en `aceptadas.csv` (política: acepta `CIERRE_FALTANTE`, `DOCUMENTOS_INCOMPLETOS`, `ANTERIOR_AL_INICIO_DE_INVENTARIO` y R5; **no** acepta `CIERRE_NO_COINCIDE_CON_ESTADO`, `PEDIDO_REPETIDO`, `PRODUCTO_NO_EN_PEDIDO` ni `SIN_INGRESO`). En BTK son 5.946 filas aceptadas y 394 pendientes para desarrollo.

### P8. Repuntar las dos rutas

No se hace con Terraform (no administra `7elgyten88`; el cambio debe revertirse en segundos). Las dos rutas se repuntan **el mismo día** (INV-027, ensayado), primero `actualizarInventario` y, solo con GO, `actualizaInventarioLotes`.

1. **Guardar la configuración actual**: id del deployment del stage, `get-integration` de cada ruta y tipo de autorización. Las dos integraciones deben ser `AWS_PROXY`; si no, parar. Anotar el timeout de integración.
2. **8a** `/actualizarInventario` (id de recurso `1cqkfp`, authorizer Cognito que se conserva) → `ApiLambdaInventarioPorGuia`: `lambda add-permission` + `apigateway put-integration` + `create-deployment`. Humo con una **guía real ya limpia** de un producto activo (sale del `acciones.csv` de P5): no debe cambiar nada.
3. **8b** `/actualizaInventarioLotes` (`nc8dn1`, sin authorizer) → `ApiLambdaInventarioPorCarga`, igual, con `Guias: [<guía>]` (un cuerpo vacío devuelve 400 a propósito).

GO: respuesta 200, el log `fin-ok` muestra `insertados=0 corregidos=0 borrados=0` y la alarma `…-PorGuia-error-en-log` sigue en OK. NO-GO: cualquier 5xx, un `E11000` sin absorber, `ERROR` en logs o `insertados>0` en una guía que debía estar limpia.

!!! warning "`create-deployment` publica todos los cambios pendientes del REST"
    Antes de publicar el stage de `7elgyten88`, confirmar que nadie dejó cambios sin publicar (por ejemplo, el webhook de novedades). Para la ruta de trazabilidad, anotar siempre el deployment anterior con `get-stage`.

## Rollback

**Regla:** se deshace en **orden inverso**; cada paso tiene el suyo.

| Si falla… | Antes de P8 (escrituras aún pausadas) | Después de P8 (el sistema nuevo ya escribe) |
|---|---|---|
| P12 front/menú | `registrar_modulo_trazabilidad --quitar` y pantalla deshabilitada | igual |
| P10/P11 | Recalcular otra vez; el resumen es derivado | igual |
| P9 barrido | Nada (simula) | Nada |
| **P8 repunte** | Volver el stage al deployment anterior (segundos), restaurar la integración hacia la gemela y `delete-function-concurrency` de esa gemela | igual (los datos limpios son compatibles con las gemelas, que insertan sin `clave`) |
| P7 despliegue | Nada que borrar: las funciones nuevas no tienen rutas de escritura | Primero repuntar las rutas a las gemelas |
| **P6 índices únicos** | `indices_golive --deshacer <registro>` (quita solo lo que creó esa corrida) | igual |
| **P5 limpieza** | `aplicar_limpieza --deshacer <carpeta>`: restaura **exactamente** lo respaldado y lo verifica documento por documento | `--deshacer` detecta conflictos (documentos que el sistema nuevo ya cambió) y no toca esos; decidir con el arquitecto si usar `--forzar`. Los datos limpios **no necesitan** deshacerse para volver a las gemelas |
| Campos de activación | `sembrar_campos_activacion_prod --deshacer <carpeta>` | igual |
| P1 pausa | `delete-function-concurrency` de las dos gemelas | n/a |

**Rollback total antes de reabrir** (probado en el ensayo): 1) quitar índices únicos; 2) deshacer la limpieza (imprime «VERIFICADO: N documentos originales iguales al respaldo y M insertados retirados»); 3) deshacer campos de activación solo si se quiere dejar todo como estaba; 4) recalcular el resumen; 5) quitar la concurrencia 0 de las gemelas; 6) correr `verificar_invariantes --fase pre`: debe reproducir la línea base de P2.

**Rollback de código:** cada ejecución del CD imprime los comandos exactos; resumen: volver a publicar el zip anterior desde S3 **por SHA** (`releases/<función>/<sha>.zip`) o desde el respaldo `respaldo/<función>/<fecha>-previo.zip`, para las funciones desplegadas. Si ya se repuntó una ruta, revertir primero el repunte.

**Desastre** (corrupción que los scripts no pueden revertir): restaurar el snapshot de P2 a un clúster nuevo y repuntar cadenas. Son horas; último recurso.

## Observabilidad

### Alarmas (22 en producción) → SNS → correo

Terraform (`modules/inventario`) crea por cada una de las **5 funciones** de producción 4 alarmas, más 2 transversales:

| Alarma | Cantidad | Qué detecta |
|---|---|---|
| `Inventario-<entorno>-<función>-errores` | 5 | Errores de ejecución de la función |
| `…-throttles` | 5 | Invocaciones rechazadas por concurrencia |
| `…-cerca-del-timeout` | 5 | Duración cercana al límite |
| `…-error-en-log` | 5 | Línea `ERROR` en el log estructurado |
| `Inventario-<entorno>-pico-tokens-rechazados` | 1 | Pico de JWT rechazados (posible ataque o integración rota) |
| `Inventario-<entorno>-barrido-cierres-faltantes` | 1 | El barrido encuentra guías cerradas sin su movimiento |

Las 22 alarmas notifican al tópico SNS `Inventario-Alarmas-<entorno>` (la suscripción por correo debe estar **confirmada**). En PreProd, con las funciones de alertas, se suma `reporte-envio-fallido` y las alarmas de esas funciones. Métricas de negocio por log: cierres conservados en retorno, barrido pendientes, envíos de reporte OK/fallidos.

### Dashboard

`Inventario-<entorno>` en CloudWatch, por Terraform: invocaciones por función, errores, duración p95, throttles, líneas `ERROR` en logs, tokens rechazados en la trazabilidad, corridas del barrido con pendientes o cierres faltantes, ejecuciones con cierres conservados en retorno (INV-038), reportes de Slack enviados/fallidos (solo donde existen las alertas) y el estado de todas las alarmas.

### Notificador (Slack)

El módulo admite reenviar las alarmas a Slack con `ApiLambdaNotificadorEventos` (dominio monitoreo) mediante la variable `notificador_slack_arn`. **Pendiente en producción:** no se verificó que exista allí, así que hoy las alarmas de producción llegan solo por correo ([Tareas futuras](../backend/inventario/tareas-futuras.md#notificador-en-produccion)). En PreProd sí está conectado.

### Logs

JSON estructurado con `correlationId` en `/aws/lambda/ApiLambdaInventario*`. Nunca llevan tokens, cadenas de conexión ni datos personales. Eventos: `entrada`, `fin-ok` (con `insertados`, `yaExistian`, `corregidos`, `borrados`, `revisar`, `repetidos`, `productosRecalculados`, `pedidosMarcados`), `validacion-fallida`, `excepcion`, `barrido-fin`, `recalculo-fin`. **Un `yaExistian` alto** = carreras absorbidas por el índice único (funciona como debe). **Un `revisar` > 0** = estados sin homologar.

## Barrido de cierre

Función `BarridoCierre` (regla EventBridge `Inventario-<entorno>-BarridoCierre-cada-3h`, `rate(3 hours)`, **creada deshabilitada y en simulación**). Recorre las guías con movimientos de los productos activos, por cursor, lee el pedido y reconcilia: agrega el cierre que falta, quita el que sobra, corrige duplicados y cantidades. Es **indispensable**: cubre los procesos que cambian el estado sin avisar al inventario (anulaciones, conciliaciones, archivadas, gestión de guías) y los cambios de estado ocultos por las banderas (hallazgo H2 del E2E).

- **`ModoSimulacion` vale `true` si no se manda**: nunca escribe por accidente. Devuelve conteos y una muestra de hasta 50 líneas.
- Deja 30 s de margen; si no alcanza, responde `Completo=false` y `SiguienteCursor`.
- **Decisión D1:** prenderlo en **simulación una semana**. Pasarlo a `ModoSimulacion=false` **solo** si la muestra de simulación fue limpia siete días seguidos **y** se decidió la regla de los cierres de guías en retorno (el reconciliador ya los conserva, [INV-038](../arquitectura/adr/inventario/ADR-INV-005-pedidosinter-fuente-de-verdad-y-retorno.md)). Luego se apagan las gemelas.
- Solo encuentra guías que **ya tienen algún movimiento**; una guía sin ninguno la registra la liquidación.

## `verificar_invariantes`

Script de solo lectura (`07-scripts-bd/verificar_invariantes.py`; **nunca escribe**). Verifica I1–I4 por producto activo, genera un CSV de detalle y uno de resumen, y devuelve un código de salida que usará el job de alertas. Opciones: `--fase pre|post`, `--aceptadas <csv>`, `--productos`, `--salidas`. Se corre: línea base en P2, GO final en P11, y **día +1 y día +7** (debe seguir en `CUMPLE`). Hoy se ejecuta a mano; automatizarlo y mandar el resultado a Slack es una mejora pendiente (decisión de salir como MVP, INV-037).

## Scripts de base de datos (resumen)

Todos en `07-scripts-bd/`, simulan por defecto y exigen `--aplicar` más una frase en producción.

| Script | Qué hace | Deshacer |
|---|---|---|
| `aplicar_limpieza.py` | Planifica, respalda, aplica con historial y verifica 0 acciones. En producción exige `--huella-esperada` | `--deshacer <carpeta>` |
| `verificar_invariantes.py` | I1–I4; nunca escribe | — |
| `indices_golive.py` | `ix_idproducto_guia`, `ix_guia`, `ix_coleccion_iddocumento`, `ux_clave`; `--probar-sparse`, `--verificar` | `--deshacer <registro>` |
| `sembrar_campos_activacion_prod.py` | `ManejaInventario`, `FechaInicioInventario`, `Tienda.ServicioInventario`; `--ids`, `--comparar-con` | `--deshacer <carpeta>` |
| `registrar_modulo_trazabilidad.py` | Documento del menú en `Modulo` | `--quitar` |
| `generar_aceptadas.py` | Propone `aceptadas.csv` y `pendientes_desarrollo.csv` (solo CSV) | — |

## Alertas diarias por Slack

El producto de suscripciones diarias tiene su propia página: [Alertas diarias por Slack](../backend/inventario/alertas-slack.md). **No** forma parte de la ventana de inventario: llega a producción después, con números ya limpios.

## Discrepancias conocidas entre documentos y código

!!! warning "Para corregir antes de la ventana"
    Las fuentes originales no coinciden en estos puntos; aquí se deja la evidencia, no se resuelve en silencio.

    | # | Discrepancia | Evidencia | Lo vigente |
    |---|---|---|---|
    | 1 | README de la lambda y runbook D-7 dicen `FORMULA_CONSUMO=v1` y `FILTRO_ACTIVACION=off` | Terraform `environments/prod/inventario/main.tf` fija `v2` y `on` (INV-034, INV-036) | **`v2` y `on`** |
    | 2 | El humo de P8 espera `Operación exitosa.` | Una guía ya limpia responde `Sin cambios: 1 pedidos leídos, 0 en Revisar` | El criterio de GO debe ser el **código 200 y `insertados=0 corregidos=0 borrados=0`**, no el texto |
    | 3 | README de la lambda dice que CORS `*` se ignora | `RespuestaHttp.cs` lo acepta; Terraform de producción lo configura (decisión 2026-10-19) | **`*`** ([ADR-INV-007](../arquitectura/adr/inventario/ADR-INV-007-cors-abierto.md)) |
    | 4 | README de la lambda: repuntar `actualizarInventario` primero y observar una semana | Runbook e INV-042 (7): las dos rutas el mismo día | **Mismo día** (recomendado y ensayado); escalonado sigue siendo una opción que decide Iker |
    | 5 | Runbook P8 dice que el repunte «no cambia» el authorizer | Es correcto: `put-integration` no modifica el método; `actualizaInventarioLotes` sigue sin authorizer | Conservar ambos como hoy |
    | 6 | `01-decisiones` tenía dos filas INV-036 e INV-037 | Renumeradas el 2026-10-19 | INV-036 = fórmula v2; INV-037 = salida MVP; INV-041/042 = decisiones de Ola 1c y Ola 2 |
