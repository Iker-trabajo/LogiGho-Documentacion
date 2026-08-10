## Autor: Iker Acevedo

Fecha creación: 2026-08-04

Estado: producción

# Sincronizacion de Conversaciones y Mensajes Pancake — Descripción general

!!! info "Pancake tiene 2 procesos automáticos independientes"
    Esta página documenta el proceso de sincronizacion de conversaciones y mensajes de todas las lineas de pancake. Existe un 2º proceso separado, con su propio Step Function y sus propias lambdas: **[Estadísticas de Cuentas (Marketing)](../overview.md)** (métricas publicitarias por campaña). No comparten código ni pipeline — solo la fuente de datos externa (Pancake).

## ¿Qué es?

Es un proceso automatizado que corre cada 3 horas (9am–6pm, hora Colombia) y mantiene en la RDS (`dbarchivoslogigho`) del area de datos una copia sincronizada de todas las conversaciones de WhatsApp/Facebook de todas las páginas activas, junto con sus **mensajes** de todas las conversaciones sincronizadas.


---

## Objetivo de proyecto

- Tener un histórico consultable por SQL de conversaciones y mensajes, para auditorías masivas y haciendo control de calidad a todas las ventas realizadas por los asesores.
- Servir de fuente para herramientas de depuración/soporte sin pegarle directo a la API de Pancake (rate limit, lentitud, no pensada para consultas masivas).

---

## Arquitectura de un vistazo

Diagrama de contexto: quién usa el sistema, con qué habla afuera, dónde persiste.

```mermaid
--8<-- "backend/pancake/conversaciones-mensajes/diagramas/contexto-c4.mmd"
```

> El detalle completo del pipeline paso a paso está en **[Orquestación](orquestacion.md)**, con su propio diagrama de estados.
>
> Fuente de este diagrama y los demás del proceso: **[carpeta de diagramas](diagramas/README.md)**.

---

## Las 2 lambdas propias de un vistazo

| # | Lambda | Rol en el pipeline | Lee / Escribe |
|---|---|---|---|
| 1 | **[ApiLambdaSincronizarConversacionesPancake](../lambdas/ApiLambdaSicronizarConversacionesPancake/ApiLambdaSincronizarConversacionesPancake.md)** | Trae las conversaciones **de 1 página** (via `Map`, 1 invocación por página) y las guarda/actualiza | Escribe `PancakeConversaciones` (MySQL) |
| 2 | **[ApiLambdaSincronizarMensajes](../lambdas/ApiLambdaSincronizarMensajes/ApiLambdaSincronizarMensajes.md)** | Toma un lote de conversaciones pendientes de **cualquier página**, trae sus mensajes en paralelo, y marca cada una como sincronizada | Escribe `PancakeMensajes` y actualiza `PancakeConversaciones` (MySQL) |

> `ApiLambdaObtenerPaginasActivas` es la **misma lambda** que usa el proceso de Estadísticas (paso 1 de ese pipeline) — se reutiliza porque ambos procesos necesitan la misma lista de páginas activas. No tiene documentación propia acá, ver la del otro proceso.

---

## Tecnologías

| Área | Tecnología |
|---|---|
| Lenguaje / arquitectura | .NET 10, Clean Architecture (Dominio / Aplicación / Infraestructura) |
| Cómputo | AWS Lambda |
| Orquestación | AWS Step Functions Standard |
| Agendamiento | Amazon EventBridge Scheduler (cron, zona `America/Bogota`) |
| Persistencia | **Aurora MySQL** (`dbarchivoslogigho`, driver `MySqlConnector`) — este proceso **no** usa MongoDB para las conversaciones/mensajes en sí (sí lee Mongo para el token de cada página) |
| Seguridad | Cadenas de conexión cifradas **AES-256-ECB** |
| API externa | Pancake (`pages.fm`) |

---

## Tablas

| Nombre | Motor | Rol |
|---|---|---|
| `PancakeConversaciones` | MySQL | 1 fila por conversación. `MensajesSincronizadosHasta` es el watermark que decide si está "al día" |
| `PancakeMensajes` | MySQL | 1 fila por mensaje, `ON DUPLICATE KEY UPDATE` por `MensajeId` |
| `PancakePaginas` / `PancakeCuentasPrincipales` | MongoDB | Origen de los tokens de página (lectura únicamente, las mantiene el otro proceso) |

---

## Convenciones clave

- **Ventana de sincronización**: `MAXIMO_DIAS_SINCRONIZAR` (env var, default `5`) — tope de profundidad histórica que trae cada lambda. Aplica tanto a qué conversaciones trae L1 (vía watermark) como a cuántos mensajes atrás pagina L2 por conversación.
- **Watermark por conversación** (`MensajesSincronizadosHasta`): si es `NULL` o más viejo que `FechaActualizacionPancake`, la conversación está "pendiente" — así el pipeline solo reprocesa lo que cambió, no todo cada vez.
- **Normalización de teléfono** (`NumeroTelefonoNormalizado`, columna generada): últimos 10 dígitos de `NumeroTelefono`, solo para filas de WhatsApp (`wa_` de prefijo) — las de Facebook llevan un PSID interno ahí, no un teléfono, y quedan `NULL` a propósito para no dar falsos positivos en auditorías.
- **Concurrencia y tokens**: dentro de una misma invocación de `ApiLambdaSincronizarMensajes` pueden correr hasta 80 conversaciones en paralelo. El token de página y el rate-limit de Pancake se coordinan a nivel de instancia (no por conversación) — ver detalle en la doc de esa lambda.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-08-04 | Iker Acevedo | Primera versión del pipeline: L1 (páginas activas, reusada) + L2 (conversaciones) + L4 (mensajes vía Step Function loop). |
| 2026-08-05 | Iker Acevedo | Se detecta y corrige loop infinito en la cola de pendientes (watermark nunca se marcaba si no había mensajes nuevos). Se agrega paralelismo (`Parallel.ForEachAsync`) e intercalado por página para evitar que 1 sola página monopolice el lote. |
| 2026-08-06 | Iker Acevedo | Se detecta y corrige condición de carrera en la regeneración de tokens de Pancake bajo alta concurrencia (múltiples conversaciones de la misma página regeneraban el token por separado y se invalidaban entre sí). Ventana bajada de 2 semanas a `MAXIMO_DIAS_SINCRONIZAR=5` días. Se agrega CloudWatch Dashboard + alarma sobre fallos del Step Function. |

---

## Deuda técnica / pendientes

- **Lambda "guardián" anti-solapamiento**: si una corrida programada (cada 3h) tarda más de 3h, la siguiente puede arrancar en paralelo sobre el mismo Step Function — hoy no hay nada que lo evite. Con los fixes de rendimiento actuales las corridas terminan en minutos, así que el riesgo es bajo, pero sigue sin resolverse de raíz.
- **Métrica custom (EMF) de "conversaciones pendientes"**: hoy para saber cuánto falta hay que consultar SQL a mano (ver [Operación](operacion.md#vistas-sql-de-monitoreo)). Falta que `ApiLambdaSincronizarMensajes` publique esa cifra como métrica de CloudWatch en cada corrida, para graficar la tendencia y alarmar si no baja.
- **Conexiones MySQL no pooleadas explícitamente**: cada repositorio abre una `MySqlConnection` nueva por llamada. Funciona, pero es una fuente probable de overhead bajo alta concurrencia — no se ha perfilado a fondo.