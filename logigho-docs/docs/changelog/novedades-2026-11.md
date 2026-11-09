# Liberación: Novedades Inter, distancia y reportes por Slack (noviembre 2026)

> Actualizado 2026-11-09: la sección «Versiones v4 a v8» al final resume lo posterior al 2026-11-06. El estado de abajo es el del 2026-11-06.

**Estado:** construida y validada en **PreProd** el 2026-11-06. **Producción: pendiente** (no hay funciones, ruta, Step ni Scheduler de esta historia en prod). El Scheduler de PreProd está apagado a propósito.

Documentación técnica: [Distancias](../backend/lambdas-dotnet/aplicacion/Interrapidisimo/ApiLambdaNovedadesInter/ApiLambdaNovedadesInterDistancias.md) · [Reportes](../backend/lambdas-dotnet/aplicacion/Interrapidisimo/ApiLambdaNovedadesInter/ApiLambdaNovedadesInterReportes.md) · [Reglas de la distancia](../backend/lambdas-dotnet/aplicacion/Interrapidisimo/ApiLambdaNovedadesInter/dominio/reglas-distancia.md) · [Reglas de los reportes](../backend/lambdas-dotnet/aplicacion/Interrapidisimo/ApiLambdaNovedadesInter/dominio/reglas-reportes.md) · [Vista](../frontend/views/logistica/novedades/novedades-distancia.md) · [Pruebas](../guias/novedades-distancia-reportes-pruebas.md).

## Nuevas funcionalidades

- **`ApiLambdaNovedadesInterDistancias`** (.NET 10): rama nueva del `Parallel` de `Step_Procesos_Logigho`. Calcula los metros por calle (en carro) entre la dirección del pedido y el punto donde el mensajero marcó cada intento, con color (≤250 verde, 251–500 amarillo, >500 rojo), estado y enlace a la ruta.
- **Regla de confianza de la geocodificación** (v3) y **confirmación del rojo** con una segunda geocodificación a ≤150 m: de 5 % a 45 % de guías calculadas sobre 277 de PreProd ([ADR-004](../backend/lambdas-dotnet/aplicacion/Interrapidisimo/ApiLambdaNovedadesInter/adr/ADR-004-regla-de-geocodificacion-v2-v3-y-confirmacion-del-rojo.md)).
- **`ApiLambdaNovedadesInterReportes`**: dos Excel (novedades a más de 500 m y guías con un intento o menos sin motivos «definitivos»), cada uno con las hojas «Nuevas desde el último reporte» y «Acumulado», subidos a Slack por el bot. Se dispara por el Scheduler (martes y viernes 8:00 Bogotá, solo prod) y por el comando `/reporte-novedades` con firma HMAC de Slack.
- **Vista `/app/logistica/novedades`:** columna de distancia con chip, filtro de 5 opciones, etiquetas «Sin dirección» y «Dirección por validar», pestañas «Mensajero / Comparar con dirección», botón «Abrir ruta en Maps» y columnas de distancia en el Excel de la vista.
- **Infraestructura como código:** 2 funciones con roles, logs y 4 alarmas, Scheduler, ruta pública del comando de Slack, rama nueva en la Step de PreProd y rol de despliegue con las 3 funciones. Un solo proyecto y un CD con lista de funciones ([ADR-003](../backend/lambdas-dotnet/aplicacion/Interrapidisimo/ApiLambdaNovedadesInter/adr/ADR-003-distancias-y-reportes-en-el-mismo-proyecto.md)).
- **Colecciones nuevas:** `ReportesNovedadesInter` y `CandadosNovedadesInter`; campos nuevos en `SeguimientoNovedadesInter` ([modelos](../backend/lambdas-dotnet/aplicacion/Interrapidisimo/ApiLambdaNovedadesInter/dominio/modelos.md)).

## Cambios que se notan

- Google omite `distanceMeters` cuando la ruta mide 0 m: la v3 lo trata como 0 m (antes quedaba como error definitivo).
- Cambiar la versión de la regla recalcula todo una vez (la versión viaja en `HuellaDistancias`).

## Pull requests mergeados

| Repo | PR | Rama destino | Tema |
|---|---|---|---|
| Lambdas_NET_LogiGho | 619 | `release/2026-11-06-novedades` | Funciones Distancias y Reportes |
| Lambdas_NET_LogiGho | 622 | `release/2026-11-06-novedades` | Feature novedades distancia y reportes |
| Lambdas_NET_LogiGho | 623 | `release/2026-11-06-novedades` | Geocodificación v2: consulta limpia, ciudad y confirmación |
| Lambdas_NET_LogiGho | 624 | `release/2026-11-06-novedades` | Ruta de 0 m es distancia válida; regla v3 |
| Terraform_Infraestructura_Logigho | 621 | `main` | Infraestructura de distancia y reportes |
| Angular SitioLogiGho | 620 | `preprod` | Vista: columna, filtro, pestañas y Excel |

**Pendiente de merge:** Angular `feature/novedades-llave-embed` (llave de navegador del mapa embebido). Los PR de Lambdas están en la rama de release; **su paso a `main` está pendiente** (no hay evidencia de que estén en `main`).

## Validación en PreProd (2026-11-06)

- 277 guías: 125 calculadas, 0 en `Error`, 38 rojas confirmadas.
- Distancias 17 s y Reportes 3 s; 14 alarmas en OK.
- Comando de Slack probado de punta a punta.

Detalle en [Pruebas](../guias/novedades-distancia-reportes-pruebas.md).

## Pendiente (producción)

Terraform de `novedades` en prod, publicar la ruta del comando en el REST, App de Slack de prod, variable group, CD con aprobación, índices (M-256), script de la Step de prod, relleno de todas las novedades, encender el Scheduler y revisión visual de Iker. Orden en el [runbook](../backend/lambdas-dotnet/aplicacion/Interrapidisimo/ApiLambdaNovedadesInter/infraestructura/runbook.md#d-historia-027-distancias-y-reportes).

## Mejoras abiertas

M-256 (índices), M-257 (lectura completa por corrida), M-258 (guías sin dirección ocupan cupo), M-259 (medir el comando en frío), M-260 (`LambdasLogiGho.preprod.sln` no compila), M-261 (bajar memoria), M-262 (variable de cadena en el plan de `datos`), M-263 (reintentar la confirmación y revisar la contención de ciudad).

## Documentación

- Páginas nuevas de las dos funciones, reglas de distancia y de reportes, ADR-003 y ADR-004, App de Slack, vista del front, guía de pruebas y variables de entorno.

## Versiones v4 a v8 (2026-11-06 a 2026-11-08)

Marcas: **observado** (ficha del dominio y código), **Iker** (fecha), **hipótesis**.

| Versión | Fecha | Qué cambia |
|---|---|---|
| v4 | 2026-11-06 | `MotivoValidacion` visible en el front; regla de geocodificación `v4` |
| v5 | 2026-11-06 | Ruta a pie (`walk1`) y distancias > 50 km a «Por revisar (más de 50 km)» |
| v6 | 2026-11-07 | Distancia = mínimo entre a pie y carro; reportes con Estado Inter, direcciones y comando `/reporte-novedades completo` y `desde AAAA-MM-DD` |
| v7 | 2026-11-07 | Reporte 1 separa «Agencia u oficina» (367 de 1 038 guías, 35 %) |
| v8 | 2026-11-07 (desplegada en prod 2026-11-08) | Dirección del mensajero, filtro de estado del reporte 1, auditoría de oficina en el reporte 2, imágenes con enlace firmado |

### Producción (2026-11-08)
- Lambdas PR 635 y Terraform PR 636. Módulo `novedades-evidencias`; el deployment del stage `Produccion` se publica por CLI (fuera de Terraform) y la ruta lleva límite burst 50 / rate 20.
- El `depends_on` se retiró del módulo del comando de Slack para que Terraform no proponga reemplazar la integración y el permiso en vivo.
- Variables nuevas: `EVIDENCIAS_SECRETO_FIRMA` (cifrada AES256), `URL_EVIDENCIAS`, `EVIDENCIAS_BUCKET`.
- Verificado: dirección del mensajero al 100 % en los Excel, enlaces con 302 (firma válida), 403 (alterada) y 400 (sin firma), 0 guías del reporte 2 en la auditoría de oficina.
- Scheduler (martes y viernes 8:00 Bogotá) **apagado** (M-271).

### Front (PR 637, 2026-11-08)
Dirección del repartidor desde el backend, mapa sin Routes API, periodo de cerradas 7/30/90/todo, cursor por `_id` (tandas de 3 000; ~4,9 s para 17 124 cerradas con 3 cursores), filtro de distancia múltiple, panel con Estado Logigho y Estado Interrapidísimo; las cifras dependen de `tiendas_asignadas`.

### Mejoras abiertas
M-264 (alarma de `CorridaCortada` y dashboard), M-267 (comparar departamento), M-268 (limitador de Routes), M-270 (carrera con la Step), M-271 (Scheduler), M-272 (costo del ciclo de 3 h), M-274 (rotación del secreto), M-275 (PreProd sin fotos), M-276 a M-280 (front).

### Documentación
[Distancias](../backend/lambdas-dotnet/aplicacion/Interrapidisimo/ApiLambdaNovedadesInter/ApiLambdaNovedadesInterDistancias.md) · [Reportes](../backend/lambdas-dotnet/aplicacion/Interrapidisimo/ApiLambdaNovedadesInter/ApiLambdaNovedadesInterReportes.md) · [Ruta de evidencias](../backend/lambdas-dotnet/aplicacion/Interrapidisimo/ApiLambdaNovedadesInter/infraestructura/evidencia-endpoint.md) · [ADR-005](../backend/lambdas-dotnet/aplicacion/Interrapidisimo/ApiLambdaNovedadesInter/adr/ADR-005-enlaces-fijos-firmados-para-las-imagenes-de-novedades.md) · [ADR-006](../backend/lambdas-dotnet/aplicacion/Interrapidisimo/ApiLambdaNovedadesInter/adr/ADR-006-direccion-del-mensajero-guardada-por-geocodificacion-inversa.md) · [Vista](../frontend/views/logistica/novedades/novedades-distancia.md) · [Pruebas](../guias/novedades-distancia-reportes-pruebas.md).
