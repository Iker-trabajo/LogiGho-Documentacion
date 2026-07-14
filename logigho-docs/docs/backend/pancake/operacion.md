## Autor: Iker Acevedo
Fecha creacion: 2026-07-13

Estado: produccion

# Operación — ejecutar, agendar y monitorear

Guía práctica para operar la integración Pancake: cómo iniciar una ejecución manual, cómo agendar nuevas franjas y cómo monitorear que la automatización esté funcionando bien.

---

## Ejecutar una toma de datos momentánea (manual)

Sirve para forzar una recolección **ahora mismo**, fuera del horario agendado — por ejemplo, un cierre puntual a las **7:00 pm**.

**Consola AWS → Step Functions → `PipelineEstadisticasPancake` → `Start execution`.**

En el cuadro **Input**, pega el JSON con la franja que quieras. Ejemplos:

Ejecución desde el inicio del día hasta el momento actual (`intradia_actual`):

```json
{
  "slot_id": "99",
  "tipo_calculo": "intradia_actual"
}
```

Ejecución del día anterior completo, desde las 00:00 hasta las 23:59 (`cierre_dia_anterior`):

```json
{
  "slot_id": "1",
  "tipo_calculo": "cierre_dia_anterior"
}
```

!!! tip "Usa un `slot_id` distinto para corridas manuales"
    Un `slot_id` como `"99"` marca la corrida como **manual/validación** y la separa de las franjas automáticas (1–4). Como el `slot_id` es parte de la clave única, no pisa los datos de las corridas oficiales y es fácil de identificar (y limpiar) en la base:

Al dar **Start execution** verás el diagrama ponerse verde estado por estado.

---

## Agregar una nueva ejecución recurrente { #agregar-nueva-ejecucion }

Para agendar una nueva franja (ej. un cuarto intradía a las 11:00 am):

**Consola AWS → Amazon EventBridge → Scheduler → Schedules → Create schedule.**

1. **Schedule name:** `pancake-slotX-11am` (nombre descriptivo).
2. **Occurrence:** Recurring schedule → **Cron-based**.
3. **Cron expression:** la hora en formato cron. Ej. 11:00 am → `cron(0 11 * * ? *)`.
4. **Timezone:** `America/Bogota` (así usas la hora local directo).
5. **Flexible time window:** Off.
6. **Target:** AWS Step Functions → `StartExecution` → `PipelineEstadisticasPancake`.
7. **Input:** el JSON de la franja:
   ```json
   { "slot_id": "5", "tipo_calculo": "intradia_actual" }
   ```
8. **Permissions:** reutiliza el rol de los otros schedules (o crea uno nuevo con `states:StartExecution`).

### Conversión de hora → cron

| Hora Colombia | Cron (con timezone `America/Bogota`) |
| ------------- | ------------------------------------ |
| 7:00 am | `cron(0 7 * * ? *)` |
| 11:00 am | `cron(0 11 * * ? *)` |
| 2:00 pm | `cron(0 14 * * ? *)` |
| 5:00 pm | `cron(0 17 * * ? *)` |
| 8:30 pm | `cron(30 20 * * ? *)` |

> Formato: `cron(min hora dia-mes mes dia-semana año)`. Usa `*` para "todos" y `?` en día-mes o día-semana (no ambos). Todo esto en formato militar. 

### Para cambiar o desactivar una franja existente

- **Cambiar la hora / el input:** EventBridge → Scheduler → el schedule → `Edit`.
- **Desactivar sin borrar:** selecciona el schedule → `Disable`.

---

## Monitorear

| Qué revisar | Dónde | Qué buscar |
| ----------- | ----- | ---------- |
| Estado de cada corrida | Step Functions → `PipelineEstadisticasPancake` → **Executions** | ~4 ejecuciones/día en verde (`Succeeded`) |
| Detalle de un fallo | La ejecución → **Graph view** | El estado que se puso rojo; ahí se lee el error |
| Logs de cada lambda | CloudWatch → Log groups → `/aws/lambda/ApiLambda...` | Los logs por etapa |
| Doble escritura activa | CloudWatch (log de `ApiLambdaObtenerEstadisticas`) | `[SqlRepo] RDS configurada -> ... HABILITADA` |
| Fallo de SQL (no bloqueante) | Idem | `[Composite] Falló SQL (no bloquea)` — si **no** aparece, SQL escribió bien |
| Datos en Mongo | MongoDB | `PancakeEstadisticasPaginas` creciendo por franja |
| Datos en RDS | Aurora MySQL | `SELECT slotId, COUNT(*) FROM PancakeEstadisticasPaginas GROUP BY slotId;` |

---

## Doble escritura a la RDS (dbarchivoslogigho) { #doble-escritura-rds }

La lambda `ApiLambdaObtenerEstadisticas` escribe en Mongo **y** en la RDS. El repositorio SQL es **plug-and-play**: arranca **inerte** si no hay cadena configurada, y se activa solo cuando existe.

### Activarla / configurarla

1. **Cifrar** la cadena de conexión con la herramienta AES:
   ```
   Server={endpoint};Port=3306;Database=dbarchivoslogigho;User ID={usuario};Password={clave};SslMode=Required
   ```
2. Poner el hex cifrado en la variable de entorno **`CADENA_CONEXION_SQL`** de la lambda.
3. *(Opcional)* Define **`TABLA_ESTADISTICAS_SQL`** solo si la tabla destino tiene otro nombre.
4. Redesplegar por terminal:
   ```powershell
   dotnet lambda deploy-function
   ```
5. Verificar que la variable quedó:
   ```powershell
   aws lambda get-function-configuration --function-name ApiLambdaObtenerEstadisticas ^
     --query "Environment.Variables" --output json
   ```

Si `CADENA_CONEXION_SQL` **no** existe, la lambda sigue funcionando escribiendo solo en Mongo.


---

## Problemas comunes y solución

| Síntoma | Causa probable | Solución |
| ------- | -------------- | -------- |
| El Step Function no aparece tras "desplegar" una lambda | `function-name` cruzado en `aws-lambda-tools-defaults.json` (copiado de otra lambda) | Verificar que cada `defaults.json` tenga **su** `function-name` y `function-handler`. Desplegar por terminal. |
| `Could not find the specified handler assembly` | Empaquetado roto (se vio con .NET 10 en este entorno) | Usar runtime **`dotnet8`** / `net8.0`, sin `PublishReadyToRun`. |
| `Lambda.TooManyRequestsException` en el Map | Throttling de concurrencia de la cuenta | Ya cubierto por el `Retry` con backoff; si persiste, bajar `MaxConcurrency` o pedir aumento de límite. |
| CloudWatch: `[SqlRepo] ... no configurada` | Falta `CADENA_CONEXION_SQL` | Agregar la variable cifrada y redesplegar. |
| CloudWatch: `[Composite] Falló SQL` | La lambda no alcanza la RDS o error de escritura | Revisar red (VPC/SG), credenciales, o anchos de columna. **No bloquea** — Mongo sigue. |
| Página con `status: "error"` | Token de la página vencido (`200` + `success:false`) | Re-sincronizar tokens (lambda 3); se corrige solo en la siguiente franja si el token se renueva. |


---

## Historial de cambios

| Fecha | Autor | Cambio |
| ----- | ----- | ------ |
| 2026-07-13 | Iker Acevedo | Creación de la guía de operación: ejecución manual, agendamiento, monitoreo, doble escritura y troubleshooting. |

---

## Observaciones

- Por el momento no hay observaciones adicionales. Se recomienda seguir monitoreando el funcionamiento del sistema en las primeras semanas de producción.
