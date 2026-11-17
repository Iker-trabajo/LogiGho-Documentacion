## Autor: Iker Acevedo

Fecha creacion: 2026-11-06  
Estado: produccion (v8 desplegada 2026-11-08; Scheduler apagado)  
Actualizado: 2026-11-09 (v6 a v8)

---

# Reglas de negocio de los reportes

Qué guías entran a cada Excel que [`ApiLambdaNovedadesInterReportes`](../ApiLambdaNovedadesInterReportes.md) sube a Slack.
Código: `Dominio/Reglas/FiltrosReporte.cs`, `FiltroEstadoReporte1.cs`, `ClasificadorAgenciaOficina.cs`, `ListaEstadosInter.cs`, `ListaMotivosExcluidos.cs`, `NormalizadorTexto.cs`, `ReglasReporte.cs`.
Decisiones: Iker 2026-11-05 (base), 2026-11-07 (v6, v7 y v8).

> Marcas: **observado** (ruta o archivo citado), **Iker** (fecha) o **hipótesis**. Esta página sigue el **código**; las diferencias con la spec 012 están en [Discrepancias](#discrepancias-con-la-spec-012).

**Universo** (Iker 2026-11-05): todo lo auditado y lo que llegue; guías `Activo` y `Cerrado` con `UltimoMotivo`.

**«Estado Inter»** (v6) es lo que ve el usuario en pantalla: `UltimaRespuesta.estado_actual` y, si viene vacío, `UltimoMotivo.EstadoInter` (`FilaReporte.EstadoInter`; en la copia de prod de 300 guías difieren en 94).
**«Estado Logigho»** es `PedidosInter.Estado` (valores observados: Intento de entrega, Devolución Regional, Devolución ratificada, Reenvío, Devolución Completada, Telemercadeo, Archivada, Pagada).

**Comparación de textos** (`NormalizadorTexto.Clave`): sin tildes, mayúsculas, espacios uniformes (también alrededor de `/`); el carácter dañado `U+FFFD` o `?` vale por **una** letra, solo en el texto que llega
(los textos de Inter llegan con tildes dañadas, observado en prod 2026-11-05). Mismo largo; un elemento de lista que traiga el comodín se descarta.

## Reporte 1: novedades a más de 500 m por calle

Archivo `novedades-mas-500m-AAAA-MM-DD.xlsx`. Entra una guía si cumple **las tres** condiciones:

1. **Distancia** (RN-7): `DistanciaUltima.Estado = Calculada` y `DistanciaUltima.Metros > 500` (el mínimo entre a pie y carro, ver [Reglas de la distancia](reglas-distancia.md)); o, para la hoja «Por revisar»,
   `BajaConfianza` con más de 50 000 m.
2. **Estado** (v8, RN-13, Iker 2026-11-07): Estado Logigho «Devolución completada» o «Devolución ratificada», **o** Estado Inter «En Proceso de Devolución». Afecta **todas** las hojas y los conteos de Slack
   (`FiltroEstadoReporte1`; listas fáciles de ampliar). Ejemplos de la spec: Inter «Envío Admitido» + Logigho «Pagada» queda fuera; «Envío Admitido» + «Devolución Completada» entra; Inter «En Proceso de Devolución» + «Archivada» entra.
3. Que no sea una guía ya separada en otra hoja (ver abajo).

- «Dirección por validar» (`BajaConfianza` por dirección, `DireccionNoEncontrada`), «Sin dirección» (`SinGeo`) y `Error` **no entran**: no prueban nada contra Inter.
- Un rojo no confirmado ya es `BajaConfianza`, así que tampoco entra.

### Hojas del reporte 1 (v7 y v8)

| Hoja | Contenido |
| ---- | --------- |
| «Nuevas desde el último reporte» | Entrega a dirección, más de 500 m, con `UltimoMotivo.Fecha` posterior a la base |
| «Acumulado» | Entrega a dirección, más de 500 m (todas) |
| «Agencia u oficina» | Acumulado de las guías de agencia u oficina (mismas columnas + «¿Nueva?» Sí/No con la misma regla de «Nuevas»). **No** entran a «Nuevas» ni a «Acumulado» |
| «Por revisar (más de 50 km)» | Entrega a dirección con ruta inverosímil (> 50 km). Las de agencia con más de 50 km van a su hoja, sin duplicarse |

Una hoja sin filas existe solo con encabezados.

### Agencia u oficina (v7, RN-7b, Iker 2026-11-07)

Medido en prod 2026-11-07: de 1 038 guías a más de 500 m, **367 (35 %)** son novedades en agencia u oficina (262 «OTROS / RESIDENTE AUSENTE (AGENCIA)» y 105 «OTROS / NO RECLAMADO EN OFICINA»),
más pedidos cuya dirección es una oficina de Inter. El mensajero no estaba en la puerta del cliente, así que la distancia a la dirección no prueba nada ante Inter y mezclarlas debilita el reclamo.
`ClasificadorAgenciaOficina`: la guía es de agencia u oficina si el último motivo **contiene** «AGENCIA» o «NO RECLAMADO EN OFICINA», **o** la dirección del pedido **contiene** «OFICINA INTERRAPIDISIMO» u «OFICINA INTER RAPIDISIMO».
**No** lo son «RESIDENTE AUSENTE (ENTREGAS)» ni «Calle 5 oficina 301». Las listas viven en código: ampliar la regla es agregar un fragmento.

## Reporte 2: un intento o menos sin motivo «definitivo»

Archivo `novedades-intentos-AAAA-MM-DD.xlsx`. Entra la guía si, **en este orden** (`FiltroReporte2`, v6 y v8):

1. **Estado Inter** dentro de `ListaEstadosInter.Reporte2PorDefecto`, hoy un solo elemento: **«En Proceso de Devolución»** (la condición más importante). Llega como «En Proceso de Devoluci�n»; «En Proceso de Devolución Total» **no** entra.
2. `PedidosInter.IntentosEntrega` es **1 o menos** (si el pedido no lo trae, no entra).
3. Tiene `UltimoMotivo.Motivo` y **no** está en la lista de exclusión (las que no tienen motivo no entran).
4. **Auditoría de oficina (v8, RN-15, Iker 2026-11-07):** se descarta toda guía cuyo número aparezca en la colección `AuditoriaCambioEntregaInter` (cualquier registro = cambió a oficina o pasó por oficina; observado en prod
   2026-11-07: 9 379 documentos, índice único `ux_numeroGuia`). Se consulta con `$in` por lotes de 1 000 y proyección solo de `NumeroGuia`, solo para las candidatas. Si la consulta falla, la corrida es **`Fallida`** y se avisa por Slack
   (mejor ningún reporte que uno con guías que ya están en oficina). Slack dice «N descartadas por oficina». Observado en prod 2026-11-08: **0** guías del reporte 2 estaban en la auditoría.

El reporte 1 **no** filtra por Estado Inter con esta lista: usa su propio filtro de estado (arriba).

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

## «Nuevas» frente a «Acumulado» y el argumento del comando

Cada Excel tiene la hoja «Nuevas desde el último reporte» y la de «Acumulado» (Iker 2026-11-05). «Nuevas» = `UltimoMotivo.Fecha` **posterior** (estricto) a la **base**:

| Base | Cómo se pide | Significado |
| ---- | ------------ | ----------- |
| `normal` | `/reporte-novedades` (o el Scheduler) | `FechaInicio` de la última corrida **Exitosa** en `ReportesNovedadesInter`; sin corrida previa, todas. Una corrida `Fallida` no mueve la frontera |
| `completo` | `/reporte-novedades completo` (también `desde-cero`) | Todas las filas son «Nuevas» |
| `desde AAAA-MM-DD` | `/reporte-novedades desde 2026-11-01` | Posteriores a las 00:00 de Bogotá de ese día; la fecha debe existir y no ser futura |

(v6, Iker 2026-11-07; `ArgumentoReporte`.) Lista blanca estricta, máximo 40 caracteres. Un argumento inválido responde la ayuda («Uso: /reporte-novedades, /reporte-novedades completo, /reporte-novedades desde 2026-11-01»)
**sin generar ni registrar nada** y sin repetir el texto recibido. El mensaje de Slack dice qué base se usó y la corrida la guarda en `BaseNuevas`. Rehacer los reportes desde cero sin tocar la base se hace con `completo`.

## Orden y columnas (v8)

- **Las hojas de guías de ambos reportes van de menor a mayor «Días desde la primera novedad»** (RN-14). El día de la primera novedad es el **1** («hoy menos la fecha de la primera novedad, más 1», días calendario de Bogotá, nunca menos de 1;
  vacío = sin fecha, al final). Primera novedad = la fecha más antigua de `UltimaRespuesta.motivos[].fecha`; si no hay, la de `Historial[].Motivo.Fecha`; si no, `UltimoMotivo.Fecha`.
  Desempate: reporte 1 por distancia descendente y luego fecha reciente; reporte 2 por fecha de la novedad descendente.
- **Columnas comunes (14):** Guía · Tienda · Seguimiento (Activo/Cerrado) · Estado Inter · Estado Logigho · Último motivo · Fecha de la novedad (hora Colombia) · Fecha de creación del pedido (hora Colombia) ·
  Días desde la creación del pedido · **Días desde la primera novedad** · Intentos · Ciudad · Dirección del pedido · **Dirección del mensajero (texto)** (`DireccionMensajero.Texto`, v8).
- **Reporte 1: 22 columnas** = las 14 + Dirección que entendió Google · Metros por calle (mínimo) · Metros a pie · Metros en carro · Metros línea recta · Ruta en Maps · Ubicación del mensajero · Ubicación del cliente.
  La hoja «Agencia u oficina» suma «¿Nueva?» (23).
- **Reporte 2: 15 columnas** = las 14 + «Ruta en Maps» («Abrir ruta», solo si es un enlace válido de Google Maps). Cierra con la hoja **«Motivos»** (una fila por cada motivo distinto de todos los seguimientos, con «Guías (todas)»,
  «Guías en este reporte» y «¿Excluido del reporte 2?», y a la derecha la tabla «Estados Inter»).
- **Imágenes (v8, RN-16, ADR-005):** después de la última columna, «Imagen novedad 1», «Imagen novedad 2», … (en «Agencia u oficina», después de «¿Nueva?»). Una por imagen, hasta el máximo de cualquier guía del reporte
  (las mismas columnas en todas las hojas), **tope duro de 10** (`EnlacesEvidencia.MaxImagenesPorGuia`; el máximo observado en prod es 6). Cada celda dice **«Ver imagen»** y es un hipervínculo al
  [enlace fijo firmado](../infraestructura/evidencia-endpoint.md). Se ordenan por `orden`, y se ignoran las imágenes sin `s3_key` o con clave que no es presignable. Sin `EVIDENCIAS_SECRETO_FIRMA` o `URL_EVIDENCIAS`
  el reporte sale **sin** esas columnas y queda la advertencia `evidencias-deshabilitadas`.
- Enlaces con texto amigable, todos HTTPS y validados (`https://www.google.com/maps...`): «Abrir ruta», «Ver mensajero» y «Ver cliente».

## Cuándo se genera

| Disparo | Detalle |
| ------- | ------- |
| EventBridge Scheduler | Martes y viernes 8:00, zona `America/Bogota` (`cron(0 8 ? * TUE,FRI *)`). **Solo producción** (Iker 2026-11-06). **Hoy apagado** (`reportes_programados_habilitados = false`; se enciende con un PR de Terraform cuando Iker lo diga, M-271) |
| Comando de Slack | `/reporte-novedades` (en PreProd se probó con `/reporte-novedades-pruebas`) y los argumentos de arriba. Solo usuarios de `SLACK_USUARIOS_PERMITIDOS`. Ver [App de Slack](../infraestructura/slack-app.md) |

El mensaje de Slack (sin datos personales) trae la base de «Nuevas», los conteos de cada reporte (reporte 1 de entrega a dirección y, aparte, «Agencia u oficina»), el top 5 por Estado Inter, Estado Logigho y motivo,
y dos validaciones reales del reporte 2 («Motivos excluidos presentes: 0 ✓», «Estados Inter fuera de la lista presentes: 0 ✓»).

## Privacidad

Desde la v6 los Excel **sí** llevan la dirección del pedido, la que entendió Google y (v8) la del mensajero: van a un **canal privado de Slack** (Iker 2026-11-07). **Nunca** llevan nombre, teléfono, identificación
ni el nombre del mensajero. Toda celda de texto se sanitiza (anti-inyección de fórmulas). Ninguna dirección se escribe en logs.

## Discrepancias con la spec 012

| Punto | Spec | Código (manda) |
| ----- | ---- | -------------- |
| Orden de las hojas del reporte 1 | §5.4 (v7): de mayor a menor distancia | RN-14 y `GeneradorExcelReportes.OrdenarReporte1`: de **menor a mayor** días desde la primera novedad, desempate por distancia descendente |
| Columnas del reporte 2 | §5.4: 13 columnas | 15 (14 comunes + «Ruta en Maps»; §5.4b de la misma spec lo dice) |
| Columnas del reporte 1 | §5.4: 21 | 22 (con «Dirección del mensajero (texto)») |
| Modo de ruta | §1 y CA1: a pie | Mínimo entre a pie y carro (v6) |
