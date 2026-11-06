## Autor: Iker Acevedo

Fecha creacion: 2026-11-06  
Estado: desarrollo (PreProd validada; producción pendiente)

---

# Reglas de negocio de los reportes

Qué guías entran a cada Excel que [`ApiLambdaNovedadesInterReportes`](../ApiLambdaNovedadesInterReportes.md) sube a Slack.
Código: `Dominio/Reglas/FiltrosReporte.cs`, `ListaMotivosExcluidos.cs`, `NormalizadorTexto.cs`. Decisiones: Iker 2026-11-05.

**Universo** (Iker 2026-11-05): todo lo auditado y lo que llegue; guías `Activo` y `Cerrado` con `UltimoMotivo`.

## Reporte 1 — novedades a más de 500 m

Archivo `novedades-mas-500m-AAAA-MM-DD.xlsx`. Entra la guía si `DistanciaUltima.Estado = Calculada` **y** `DistanciaUltima.Metros > 500`.

- «Dirección por validar» (`BajaConfianza`, `DireccionNoEncontrada`), «Sin dirección» (`SinGeo`) y `Error` **no entran**: no prueban nada contra Inter.
- Un rojo no confirmado ya es `BajaConfianza`, así que tampoco entra (ver [Reglas de la distancia](reglas-distancia.md#5-confirmacion-del-rojo)).

## Reporte 2 — un intento o menos sin motivo «definitivo»

Archivo `novedades-intentos-AAAA-MM-DD.xlsx`. Entra la guía si:

- tiene `UltimoMotivo.Motivo` (las que no tienen motivo no entran),
- `PedidosInter.IntentosEntrega` es **1 o menos** (si el pedido no lo trae, no entra), y
- su motivo **no** está en la lista de exclusión.

### Motivos excluidos (Iker 2026-11-05)

| # | Motivo |
| - | ------ |
| 1 | `REHUSADO / YA NO DESEA EL ENVIO` |
| 2 | `DIRECCION ERRADA / DIRECCION INCOMPLETA` |
| 3 | `DIRECCION ERRADA / DIRECCION NO EXISTE` |
| 4 | `REHUSADO / NO CORRESPONDE A LO SOLICITADO` |
| 5 | `DIRECCION ERRADA / NO CORRESPONDE LA CIUDAD DESTINO` |
| 6 | `REHUSADO / NO SE PERMITE APERTURA DEL ENVIO` |
| 7 | `REHUSADO / NO SOLICITO EL ENVIO` |
| 8 | `REHUSADO / VALOR COBRADO NO CORRESPONDE AL PACTADO` |

La lista es configurable con `MOTIVOS_EXCLUIDOS_REPORTE2` (separada por `;`); cambiarla es correr el CD, no tocar código.
Variable vacía o sin elementos útiles: se usan los 8 por defecto.

### Normalización de los motivos

Los textos de Inter llegan con tildes dañadas (`U+FFFD`, observado en prod 2026-11-05) y con espacios irregulares alrededor de la barra.
Se compara **clave con clave** (`NormalizadorTexto.Clave`):

1. Se quitan las tildes (descomposición NFD sin marcas).
2. `U+FFFD` y `?` se tratan como **comodín de un carácter**.
3. Mayúsculas, espacios colapsados y espacios uniformes alrededor de `/`.

Ejemplo: `REHUSADO /NO SOLICITO EL ENVIO` y `REHUSADO / NO SOLICITO EL ENVIO` son la misma clave. Reglas de seguridad de la comparación:
mismo largo, y el comodín **solo vale en el texto que llega**; un elemento de la lista que lo traiga se descarta.

## «Nuevas» frente a «Acumulado»

Cada Excel tiene dos hojas (Iker 2026-11-05):

| Hoja | Contenido |
| ---- | --------- |
| «Nuevas desde el último reporte» | Filas del reporte cuyo `UltimoMotivo.Fecha` es **posterior** a la frontera. En la primera corrida (sin corrida previa), todas |
| «Acumulado» | Todas las filas del reporte |

**Frontera:** el `FechaInicio` de la última corrida **Exitosa** en `ReportesNovedadesInter`. Una corrida `Fallida` no mueve la frontera,
así la siguiente no pierde las «Nuevas». (Observado: `GenerarReportesUseCase.cs`.)

## Cuándo se genera

| Disparo | Detalle |
| ------- | ------- |
| EventBridge Scheduler | Martes y viernes 8:00, zona `America/Bogota` (`cron(0 8 ? * TUE,FRI *)`). **Solo producción** (Iker 2026-11-06); en PreProd queda apagado. En prod nace apagado: **pendiente** encenderlo en el paso de salida |
| Comando de Slack | `/reporte-novedades` (en PreProd se probó con `/reporte-novedades-pruebas`). Solo usuarios de `SLACK_USUARIOS_PERMITIDOS`. Ver [App de Slack](../infraestructura/slack-app.md) |

## Privacidad

Los Excel llevan guía, tienda, estados, motivo, fechas, ciudad, metros y enlaces. **No** llevan nombre, teléfono ni dirección del destinatario.
