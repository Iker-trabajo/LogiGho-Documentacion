# Pruebas de novedades: distancia y reportes (historia 027)

Plan de pruebas CA1 a CA14 de la spec 012 y el estado observado en PreProd el 2026-11-06, más los criterios de las versiones v6 a v8 (CA15 a CA23) con lo medido en producción el 2026-11-08. Sin datos personales: solo conteos y resultados agregados.

!!! note "Qué significa cada estado"
    **Observado** = la fuente citada lo muestra. **Parcial** = hay evidencia de una parte. **Pendiente** = no se ha probado o depende de producción.
    Fuente de los resultados: `tasks.md` y `spec.md` de `specs/012_novedades_distancia_reportes/` (Iker, validación del 2026-11-06).

## Resultados agregados en PreProd (2026-11-06)

| Dato | Valor |
| ---- | ----- |
| Guías de muestra (copiadas de prod, solo campos necesarios; 300 seguimientos y 300 pedidos el 2026-11-05) | 277 guías evaluadas |
| Distancias calculadas | 125 de 277 (45 %); con la regla inicial era 5 % ([ADR-004](../backend/lambdas-dotnet/aplicacion/Interrapidisimo/ApiLambdaNovedadesInter/adr/ADR-004-regla-de-geocodificacion-v2-v3-y-confirmacion-del-rojo.md)) |
| Guías en `Error` | 0 |
| Rojas confirmadas (segunda geocodificación) | 38 |
| Duración y memoria de Distancias | 17 s, ~136 MB de 512 |
| Duración y memoria de Reportes | ~3 s, ~157 MB de 1024 |
| Alarmas de CloudWatch en OK | 14 de 14 |
| Comando de Slack | Probado de punta a punta (`/reporte-novedades-pruebas`) |
| Scheduler de PreProd | Apagado a propósito (solo prod) |
| Pruebas automáticas en la rama del 2026-11-05 | 393 tests .NET, 27 Python, 81 specs de Angular, 5 tests de Terraform (`tasks.md`) |

## Plan CA1 a CA14

| CA | Qué se prueba | Cómo | Estado (PreProd 2026-11-06) |
| -- | ------------- | ---- | --------------------------- |
| CA1 | Guía con coordenada y dirección confiable: `DistanciaUltima.Estado = Calculada`, metros iguales a Maps en carro (±10 %), `UrlRuta` válida | Elegir 3 guías calculadas, abrir `UrlRuta` y comparar con Maps | 125 calculadas; **comparación contra Maps: revisión de Iker pendiente** |
| CA2 | Intento sin coordenada: `SinGeo` y «Sin dirección» en la vista | Buscar un seguimiento sin `Geo` y verlo en la vista | Cubierto por tests de dominio y front; **confirmación visual pendiente** |
| CA3 | `APPROXIMATE`/`GEOMETRIC_CENTER`: `BajaConfianza`, «Dirección por validar», fuera del reporte 1 | Contar `BajaConfianza` y comprobar que no están en el Excel | Regla probada con tests; **revisión del Excel pendiente** |
| CA4 | Colores: 250 verde, 251 amarillo, 500 amarillo, 501 rojo | Tests `ReglasDistanciaTests` y `novedades.rules.spec.ts` | Observado en tests |
| CA5 | Segunda corrida sin cambios: 0 llamadas a Google | Correr Distancias dos veces; mirar `LlamadasGoogle` | Cubierto por test del caso de uso; **medición en PreProd pendiente de evidencia** |
| CA6 | Si Distancias falla, el resto de la Step termina | Forzar un error y ver la ejecución de la Step | Rama con `Catch` hacia el fin (observado en el `.tftpl`); **ejecución forzada pendiente** |
| CA7 | El reporte 2 excluye los 8 motivos aunque lleguen con tildes dañadas (`U+FFFD`) o espacios distintos | Tests de `NormalizadorTexto` y `GenerarReportesUseCase` | Observado en tests |
| CA8 | Cada Excel tiene las hojas «Nuevas desde el último reporte» y «Acumulado», las columnas definidas y ningún dato personal | Abrir los dos archivos recibidos en Slack | Archivos recibidos en el canal de prueba; **revisión de columnas por Iker pendiente** |
| CA9 | Una celda que empieza por `=` sale como texto | Test `ExcelTests` | Observado en tests |
| CA10 | El comando responde en menos de 3 s y en menos de 2 min llegan los 2 archivos; firma inválida da 401 y usuario no permitido da 403 | Ejecutar el comando; prueba con firma alterada | Comando probado de punta a punta; Reportes tarda ~3 s. **Medición con función fría pendiente (M-259)** |
| CA11 | El Scheduler corre martes y viernes a las 8:00 de Bogotá | Validar la expresión en Terraform y una corrida forzada | Expresión `cron(0 8 ? * TUE,FRI *)` en `America/Bogota` (observado); **apagado en PreProd; en prod, pendiente** |
| CA12 | La vista muestra columna, filtro de 5 opciones, pestaña «Comparar con dirección» con «Abrir ruta en Maps» y distancia en el Excel de la vista | Abrir `/app/logistica/novedades` en PreProd | Hecho en el código ([vista](../frontend/views/logistica/novedades/novedades-distancia.md)); **revisión visual de Iker pendiente** |
| CA13 | Ningún log contiene dirección, nombre, teléfono, llave ni token | Revisión de código y búsqueda en CloudWatch de PreProd | Revisor independiente aprobó con cambios el 2026-11-05 (corregidos); **revisión específica de logs y búsqueda en CloudWatch: sin evidencia en las fuentes** |
| CA14 | `dotnet test`, `ng test` y `terraform validate`/`plan` sin destrucciones inesperadas | CI | Tests en verde el 2026-11-05; `terraform apply` de PreProd realizado. `LambdasLogiGho.preprod.sln` no compila por una referencia rota previa (M-260) |

## Pruebas manuales útiles

1. **Distancias a mano:** Step Functions, `Start execution` con `{}` en PreProd, o invocar `ApiLambdaNovedadesInterDistancias` con `{"origen":"step"}`. Leer el log `[RESULT] fin` (conteos).
2. **Idempotencia (CA5):** repetir y comprobar `llamadasGoogle = 0`.
3. **Reportes:** ejecutar el comando de Slack con un usuario permitido; verificar el mensaje inmediato y los 2 archivos.
4. **Seguridad del comando:** firma alterada, `X-Slack-Request-Timestamp` viejo (más de 300 s) y usuario fuera de `SLACK_USUARIOS_PERMITIDOS` deben dar 401, 401 y 403.
5. **Prod antes de liberar (pendiente):** probar en solo lectura (`explain`) la selección de pendientes y el `$set` sobre DocumentDB, porque el éxito en Atlas no prueba DocumentDB.

## Riesgo principal y criterio de parada

Calidad de la geocodificación en Colombia (spec 012 §8). El criterio original de la spec era parar y replantear con Iker si menos del 60 % salía confiable. Con la regla inicial salió 5 % y con la v2/v3, 45 %; la regla nueva y la confirmación del rojo quedaron registradas como decisión de Iker el 2026-11-06. **Hipótesis:** el 45 % queda por debajo del 60 % de la spec; conviene dejar escrito que Iker lo aceptó. La revisión de las guías «Dirección por validar» sigue siendo una decisión de negocio.

## Versiones v6 a v8: CA15 a CA23 y lo observado en producción (2026-11-08)

Fuente: spec 012 §7 y `docs/architecture/dominios/novedades.md` del workspace (verificación del 2026-11-08). Los casos de código se prueban con tests de `ApiLambdaNovedadesInter.Tests` (711 pruebas el 2026-11-07 según la ficha del dominio; los nombres de clase por criterio son los de la spec).

!!! note "Verificado el 2026-11-09"
    Las clases de test de la historia 027 (por ejemplo `FirmaEvidenciaTests`) sí están en `origin/main` de `Lambdas_NET_LogiGho` (PR 635, commit `993ee006`); se comprobó con `git ls-tree` sobre `origin/main`. La suite de la rama `feature/novedades-reportes-v8` pasó 958 de 958 antes del merge (observado por Iker el 2026-11-07/08).


| CA | Qué se prueba | Cómo | Estado |
| -- | ------------- | ---- | ------ |
| CA15 (v6) | Mínimo entre modos: caso Pailitas (a pie 38 561, carro 791/973, recta 593) queda en 791 m `Calculada`, roja y fuera de «Por revisar»; a pie ≤ 250 m no pide carro; una guía con 900 m a pie y sin carro vuelve a pendiente sin geocodificar ni repetir la ruta a pie | Tests de dominio y del caso de uso; medición en prod con 1 060 rojas (solo 4 dejan de serlo) | Observado en tests y en la medición de prod 2026-11-07 |
| CA16 (v6) | Reporte 2 solo con Estado Inter «En Proceso de Devolución» (con el carácter dañado), ≤ 1 intento y motivo no excluido; hoja «Motivos» al final; Slack con «0 ✓» en las dos validaciones | Tests de `GenerarReportesUseCase`, `ReglaHojaMotivos` | Observado en tests (el resultado en prod se vio en los Excel del 2026-11-08) |
| CA17 (v6) | `/reporte-novedades completo` y `desde AAAA-MM-DD` cambian «Nuevas» y lo dicen en el mensaje; argumento inválido responde la ayuda sin generar ni registrar | Tests de `ArgumentoReporte` y del handler | Observado en tests |
| CA18 (v8) | `DireccionMensajero`: `Calculada` con texto; segunda corrida sin llamadas; `SinResultado` no se reintenta con la misma `HuellaGeo`; `Error` sí; punto nuevo se recalcula; guía ya al día se completa sin rutas ni geocodificar el pedido; texto nunca en logs | `DireccionMensajeroTests`, `ClienteGoogleMapsInversaTests` | **Prod 2026-11-08 (observado):** 8 112 `Calculada`, 0 `SinResultado`, 0 `Error`; ~8 100 llamadas, ~USD 40 una vez |
| CA19 (v8) | Reporte 1 solo con Estado Logigho «Devolución completada/ratificada» o Estado Inter «En Proceso de Devolución» en las 4 hojas | `FiltroEstadoReporte1Tests`, `GenerarReportesUseCaseTests` | Observado en tests |
| CA20 (v8) | Hojas de guías de ambos reportes de menor a mayor «Días desde la primera novedad»; columna «Dirección del mensajero (texto)» en ambos y ruta en el reporte 2 | `GeneradorExcelV8Tests` | Observado en tests; en prod los Excel trajeron la dirección del mensajero al **100 %** |
| CA21 (v8) | Una guía con registro en `AuditoriaCambioEntregaInter` no está en el reporte 2; consulta por lotes solo de candidatas; Slack «N descartadas por oficina»; si la consulta falla, corrida `Fallida` | `AuditoriaOficinaReporte2Tests` | Observado en tests; en prod **0** guías del reporte 2 estaban en la auditoría |
| CA22 (v8) | Una columna «Imagen novedad N» por imagen (máx. 10) con «Ver imagen» al enlace firmado; sin secreto o URL el reporte sale sin esas columnas | `GeneradorExcelV8Tests` | Observado en tests; en prod los Excel traen los enlaces |
| CA23 (v8) | Endpoint: firma válida 302 a URL prefirmada de 300 s; firma inválida, clave con `..`, prefijo o extensión raros 403 genérico; sin parámetros 400; `Cache-Control: no-store`; sin secretos en logs; el generador y el validador comparten `Firmar` | `FirmaEvidenciaTests`, `ResolverEvidenciaUseCaseTests`, `EvidenciaHandlerTests`, `PresignadorS3EvidenciasTests` | **Prod 2026-11-08 (observado):** 302 a S3 con firma válida, 403 con firma alterada, 400 sin firma |

### Pruebas manuales de la v8 (sin datos personales)

1. **Dirección del mensajero:** ejecutar Distancias dos veces (`{}`) y comprobar en el `[RESULT] fin` que la segunda da `llamadasGoogle = 0`. Contar en Mongo (script de lectura, con `--proyeccion`) los `DireccionMensajero.Estado` por valor, sin imprimir textos.
2. **Enlace firmado:** abrir un «Ver imagen» de un Excel reciente (debe abrir la foto), alterar un carácter de `f` (403) y quitar `f` (400). No pegar enlaces ni la firma en chats ni en `docs/`.
3. **Filtros del reporte 1:** comprobar que las 4 hojas solo traen guías con los estados permitidos (columnas «Estado Inter» y «Estado Logigho»).
4. **Reporte 2:** confirmar que Slack dice «Motivos excluidos presentes: 0 ✓», «Estados Inter fuera de la lista presentes: 0 ✓» y el número de descartadas por oficina.
5. **Comando con argumento:** `/reporte-novedades completo` y `/reporte-novedades desde 2026-11-01` (el mensaje dice la base); un argumento inválido responde la ayuda.
6. **Front (PR 637):** abrir `/app/logistica/novedades`, elegir «Todo el historial» (medido ~4,9 s para 17 124 cerradas con 3 cursores), combinar dos opciones de distancia y abrir el detalle: ver Estado Logigho, Estado Interrapidísimo, dirección del repartidor y el mapa con línea recta sin pedidos a Routes (revisar la pestaña de red del navegador).

### Pendiente de pruebas y evidencia

- PreProd no puede leer las fotos del bucket de prod (otra cuenta): el flujo completo de imágenes solo se probó en prod (M-275).
- Cobertura del front: 81 specs eran las del PR 620; el número vigente tras el PR 637 **no se verificó** en esta documentación.
- El Scheduler (martes y viernes 8:00) está apagado en prod: CA11 sigue sin corrida real (M-271).
