## Autor: Iker Acevedo
Fecha creación: 2026-08-07

Estado: producción

# Operación — ejecutar, monitorear y auditar

Guía práctica: cómo correr una sincronización manual, cómo monitorear con CloudWatch, las vistas SQL para ver el estado de la cola, y los problemas más comunes que ya se dieron en producción.

---

## Ejecutar una sincronización manual

**Consola AWS → Step Functions → `Step_Conversaciones_Pancake` → `Start execution`.**

Input: `{}` (la 1ª lambda no necesita nada propio).

Es seguro correrla manualmente aunque haya una ejecución agendada corriendo — no hay problema de datos corruptos, solo el riesgo de rendimiento mencionado en [Orquestación](orquestacion.md#agendamiento-eventbridge-scheduler).

---

## Redesplegar tras un cambio de código

`dotnet lambda deploy-function` **no interrumpe** una invocación en curso — la que ya arrancó termina con el código viejo (el entorno de ejecución ya estaba levantado); la **siguiente** invocación (la próxima vuelta del self-loop, o la próxima corrida agendada) usa el código nuevo automáticamente. No hace falta pausar el Step Function para desplegar.

```powershell
cd LambdasLogiGho.Aplicacion/Pancake/ApiLambdaSincronizarMensajes/ApiLambdaSincronizarMensajes
dotnet lambda deploy-function --config-file aws-lambda-tools-defaults.preprod.json
```

Mismo comando para `ApiLambdaSincronizarConversacionesPancake`, desde su propia carpeta.

---

## Monitoreo — CloudWatch

### Dashboard

`Pancake-Sincronizacion` (CloudWatch → Dashboards). Widgets:

| Widget | Métrica | Para qué |
|---|---|---|
| Errores de las 2 lambdas | `Lambda` → `Errors` (Sum) de `ApiLambdaSincronizarMensajes` y `ApiLambdaSincronizarConversacionesPancake` | Ver si alguna lambda está fallando de forma sostenida |
| Step Function: éxitos vs fallos | `States` → `ExecutionsSucceeded` / `ExecutionsFailed` de `Step_Conversaciones_Pancake` | Ver si alguna de las 4 corridas diarias falló completa |
| Duración de cada corrida | `States` → `ExecutionTime` | Detectar si el self-loop se está alargando de más |
| Duration (Lambda) | `Lambda` → `Duration` de ambas lambdas | Detectar cold starts o lentitud puntual |
| Invocaciones | `Lambda` → `Invocations` de ambas lambdas | Ver el ritmo real de las 4 corridas/día |

### Alarma

`Pancake-StepFunction-Fallo` — dispara si `ExecutionsFailed > 0` (1 solo fallo, `Sum`, período 5 min, `1 out of 1` datapoints). Notifica por email vía SNS.

> Si el email no llega: revisar que se haya confirmado la suscripción de SNS (el correo "AWS Notification - Subscription Confirmation").

### Logs

CloudWatch → Log groups → `/aws/lambda/ApiLambdaSincronizarMensajes` y `/aws/lambda/ApiLambdaSincronizarConversacionesPancake`.

Buscar la línea de resumen de cada lote:
```
Lote de mensajes pendientes OK. procesadas=X, conError=Y, mensajes=Z, cortoPorTiempo=W.
```
Si `conError` es una fracción grande de `procesadas` (ej. >50%), sospechar de tokens de Pancake vencidos en masa — ver [Problemas comunes](#problemas-comunes-y-solucion).

---

## Vistas SQL de monitoreo { #vistas-sql-de-monitoreo }

Corridas contra `dbarchivoslogigho`. Crear 1 sola vez, consultar después con `SELECT * FROM nombre_vista;`.

```sql
-- Avance global
CREATE VIEW VW_EstadoSincronizacionPancake AS
SELECT
  COUNT(*) AS total_conversaciones,
  SUM(CASE WHEN MensajesSincronizadosHasta IS NOT NULL THEN 1 ELSE 0 END) AS sincronizadas,
  COUNT(*) - SUM(CASE WHEN MensajesSincronizadosHasta IS NOT NULL THEN 1 ELSE 0 END) AS pendientes,
  ROUND(100.0 * SUM(CASE WHEN MensajesSincronizadosHasta IS NOT NULL THEN 1 ELSE 0 END) / COUNT(*), 1) AS pct_avance
FROM PancakeConversaciones;

-- Avance por página (para ver si el atraso está concentrado en 1 cuenta)
CREATE VIEW VW_EstadoSincronizacionPorPagina AS
SELECT
  PageId,
  COUNT(*) AS total,
  SUM(CASE WHEN MensajesSincronizadosHasta IS NOT NULL THEN 1 ELSE 0 END) AS sincronizadas,
  COUNT(*) - SUM(CASE WHEN MensajesSincronizadosHasta IS NOT NULL THEN 1 ELSE 0 END) AS pendientes
FROM PancakeConversaciones
GROUP BY PageId
ORDER BY pendientes DESC;
```

Para sacar un ritmo real (conversaciones/minuto): correr `SELECT * FROM VW_EstadoSincronizacionPancake;` 2 veces con ~10 min de diferencia y comparar `sincronizadas`.

---

## Auditoría de números de teléfono

Herramienta aparte (Python, fuera de este repo): `Herramientas - desarrollo\Depurador-conversaciones-pancake`. Recibe una lista de números (`.txt`/`.csv`/`.xlsx`) y dice, por cada uno, si está sincronizado, pendiente, o no existe en Pancake — sin pegarle a la API de Pancake directo. Documentación de uso en el `README.md` de ese proyecto.

---

## Problemas comunes y solución

| Síntoma | Causa | Solución |
|---|---|---|
| Cola de pendientes no baja / crece | El código desplegado no tiene los fixes de paralelismo (verificar con `LlamadasObtenerMensajes` vs conversaciones procesadas en logs) | Verificar que ambos `aws-lambda-tools-defaults*.json` tengan `MAXIMO_DIAS_SINCRONIZAR` y `LIMITE_CONVERSACIONES_PENDIENTES` correctos, redesplegar |
| `conError` muy alto en el log de resumen (ej. 700+/800), todos con `error_code=105` (`access_token renewed`) | Condición de carrera regenerando el token de una página muy activa bajo alta concurrencia | Corregido en el fix de agosto 2026 (semáforo + caché de token por página, ver historial en [Visión general](overview.md)) — si vuelve a aparecer, revisar que el fix siga desplegado |
| `States.DataLimitExceeded` en el Step Function | Se intentó pasar una lista grande de pendientes por el estado del Step Function (regresión al diseño viejo) | La lambda de mensajes **nunca** debe recibir la lista de pendientes por input — debe autoconsultar MySQL. Ver [Orquestación](orquestacion.md#por-que-el-self-loop-y-no-un-map) |
| Backlog histórico enorme tras un cambio de ventana | Se redujo `MAXIMO_DIAS_SINCRONIZAR` pero las conversaciones viejas ya estaban en la cola (`MensajesSincronizadosHasta IS NULL`) | Se puede "saltar" el histórico viejo marcándolo como sincronizado sin traer mensajes (`UPDATE PancakeConversaciones SET MensajesSincronizadosHasta = NOW() WHERE MensajesSincronizadosHasta IS NULL AND FechaActualizacionPancake < (NOW() - INTERVAL N DAY)`) — sacrifica profundidad histórica en la copia local a cambio de ponerse al día rápido |
| Rate limit de Pancake (429) | Muchas conversaciones de la misma página en paralelo sin coordinación | `PancakeMensajesClient` ya limita a 3 req/s por página (margen bajo el límite real de 5/s) vía semáforo estático por `pageId` |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-08-07 | Iker Acevedo | Creación de la guía de operación: ejecución manual, redeploy sin downtime, dashboard + alarma de CloudWatch, vistas SQL de avance, herramienta de auditoría, troubleshooting. |

---

## Observaciones

- A diferencia del proceso de Estadísticas, acá **no hay `slot_id`** ni distintos tipos de corrida — cada ejecución hace lo mismo (drenar lo pendiente), agendada o manual.
- El corte por tiempo dentro de `ApiLambdaSincronizarMensajes` (margen de 30s antes del timeout de Lambda) es lo que hace posible que el self-loop sea seguro: nunca deja una invocación a medio commit, simplemente para y deja el resto para la próxima vuelta.
