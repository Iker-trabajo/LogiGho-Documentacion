---

## Autor: Iker Acevedo  
Fecha creacion: 2026-11-09  
Estado: aceptada

# ADR-006 — Dirección del mensajero por geocodificación inversa guardada, una vez por punto

**Autor:** Iker Acevedo  
**Fecha:** 2026-11-09 (decisiones de Iker 2026-11-07 y 2026-11-08; desplegada en producción 2026-11-08)  
**Estado:** Aceptada

> Marcas: **observado** (ruta o fuente citada), **Iker** (fecha) o **hipótesis**.

---

## Contexto

ws-inter solo entrega al mensajero como coordenadas: `UltimoMotivo.Geo {Latitud, Longitud, Url}` (observado en prod 2026-11-07: 8 114 de 18 051 seguimientos traen `Geo.Latitud`). La gerencia
quiere **leer la dirección** donde se marcó la novedad, tanto en los Excel como en la vista, para reclamarle a Inter sin abrir un mapa por cada guía (spec 012 RN-12).

Hasta la v7 la vista del front obtenía la dirección con la geocodificación inversa de la **llave de navegador** (cada usuario, en memoria, cada vez que abría el detalle) y los Excel no la traían.

## Opciones consideradas

### Opción A — Calcularla en el navegador (como estaba)

La vista llama a la API de Google con la llave de navegador y guarda el resultado en memoria de la pestaña (`helpers/direccion-inversa.ts`).

**Pros:** no hay cambios de backend.  
**Contras:** **cada usuario y cada apertura paga una llamada** (la caché vive solo mientras la pestaña está abierta); los **Excel no pueden** traer la dirección (se generan en el backend);
el resultado no es estable entre personas; depende de que la llave de navegador tenga la API habilitada y de sus cuotas.

### Opción B — Calcularla al generar cada reporte

Reportes llama a Google por cada guía cada vez que arma el Excel.

**Pros:** sin campos nuevos en Mongo.  
**Contras:** se repite el cobro en cada reporte (martes, viernes y cada comando); aumenta la duración del comando de Slack, que debe responder rápido (CA10); el Excel y la vista pueden mostrar textos distintos.

### Opción C — Geocodificación inversa en Distancias, guardada en Mongo, una vez por guía y por punto

La función Distancias llama a `geocode/json?latlng=lat,lng&language=es` y guarda
`SeguimientoNovedadesInter.DireccionMensajero {Texto, HuellaGeo, FechaCalculo, Estado: Calculada|SinResultado|Error}`. Reportes y el front **leen** el campo, sin llamar a Google.

**Pros:** se paga **una vez por punto** (la `HuellaGeo` es el punto a 6 decimales; solo se repite si el mensajero marca otro); Excel y vista muestran el mismo texto; los reportes no dependen de Google;
`SinResultado` no se reintenta mientras la huella no cambie; `Error` sí.  
**Contras:** un campo más (dato personal derivado de la ubicación: nunca se loguea); un cobro inicial por el relleno de lo ya procesado; si el punto no tiene calle, Google puede devolver un barrio (mejora **M-272**: evaluar `result_type`).

## Costos

| Opción | Costo | Observación |
| ------ | ----- | ----------- |
| A | Una llamada **por usuario y por apertura** (sin tope previsible) | Es el cobro que Iker detectó el 2026-11-08 al revisar el front |
| B | Una llamada **por guía y por reporte** | Crece con cada reporte y con cada argumento del comando |
| C | Una llamada por guía y por punto; **relleno único observado 2026-11-08: ~8 100 llamadas, ~USD 40** (≈ USD 0.005 por llamada, cifra derivada de esos dos datos) y luego solo las guías nuevas | Medido en prod: 8 112 `Calculada`, 0 `SinResultado`, 0 `Error` |

## Decisión

**Se eligió:** Opción C (Iker 2026-11-07, reportes v8; y 2026-11-08 para el front).

**Razón:** menor costo total, un solo texto para Excel y vista, y ninguna dependencia de Google al generar un reporte. Además **no sube ninguna versión de regla**
(`VersionReglaGeocodificacion = v4`, `VersionReglaRuta = walk1`): la dirección del mensajero se completa con una marca independiente (`ReglaDireccionMensajero.Necesita`), así que lo ya calculado no se recalcula.

**Front (Iker 2026-11-08):** la vista usa `DireccionMensajero.Texto` cuando su `HuellaGeo` coincide con el último punto del intento (costo USD 0). Para intentos **anteriores** se mantiene la
geocodificación del navegador, porque el backend solo guarda el punto del último intento.

## Consecuencias

**Positivas:** la columna «Dirección del mensajero (texto)» salió al 100 % en los Excel de prod 2026-11-08; reprocesar es repetir las corridas hasta vaciar pendientes (ver
[runbook D8](../infraestructura/runbook.md#d8-reprocesar-por-tandas-backfill)).  
**Negativas:** hay que vigilar el costo del ciclo de 3 h (mejora M-272); un `Error` de servicio cuenta para cortar la corrida (`FALLAS_SEGUIDAS_PARA_CORTAR`); carrera conocida con la Step: las guías nuevas pueden
esperar hasta la corrida siguiente, hasta 3 h (mejora M-270; la causa es **hipótesis**).

## Impacto en el código

| Módulo / Repo | Cambio |
| ------------- | ------ |
| `Lambdas_NET_LogiGho` | `ClienteGoogleMaps.GeocodificarInversaAsync`, `ReglaDireccionMensajero.cs`, entidad `DireccionMensajero`, `DistanciasRepositorio.GuardarDireccionMensajeroAsync` (`$set` de un solo campo), `CalcularDistanciasUseCase` (PR 635) |
| `Angular SitioLogiGho` | `novedades.mapper.ts` (`direccionMensajeroDesdeDoc`) y proyección en `novedades.repository.ts` (PR 637) |

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-11-09 | Iker Acevedo (redactado con IA) | Creación a partir de las decisiones del 2026-11-07 y 2026-11-08 y la medición en prod |

## Referencias

- [Reglas de la distancia](../dominio/reglas-distancia.md#9-direccion-del-mensajero-v8) · [Modelos](../dominio/modelos.md) · [Distancias](../ApiLambdaNovedadesInterDistancias.md) · [ADR-005](ADR-005-enlaces-fijos-firmados-para-las-imagenes-de-novedades.md)
- Spec 012 RN-12 y RN-12b; decisiones 2026-11-07 y 2026-11-08 (`docs/architecture/decisiones.md`); M-270, M-272.
