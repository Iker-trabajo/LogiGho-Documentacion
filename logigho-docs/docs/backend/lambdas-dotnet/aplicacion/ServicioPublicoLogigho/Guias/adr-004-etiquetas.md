---
autor: Iker Acevedo
fecha_creacion: 2026-10-22
estado: aceptada
---

# ADR-004 — Etiquetas: el worker las genera y escribe directo en los buckets actuales

**Fecha:** 2026-10-22
**Estado:** Aceptada

---

## Contexto

Hoy existen dos piezas relacionadas, que no son alternativas del mismo nivel:

| | `Step_Creacion_Etiqueta` → `ApiLambdaCrearEtiquetaManual` | `PUT /putObject` → `ApiLambdaPutObjectAOT` |
| --- | --- | --- |
| Qué hace | Proceso completo: pide la etiqueta a la transportadora, guarda `.txt` y PDF (llamando a `/putObject`) y actualiza `UrlGuia` | Solo guarda en S3 |
| Datos prod (30 días) | ~8 s promedio, máx. 161 s | 73 ms promedio, 119k invocaciones, 0 errores |
| Errores | Un error por guía vuelve como texto `"Error ..."` con éxito, así que la alarma SNS casi nunca suena | Claros |
| Requisitos | Token Cognito de usuario en el body | Ninguno: la ruta no tiene authorizer ni API key |
| Transportadoras | 6 lógicas, con bugs (D2E sin `await`, TCC rechazado) | Lista fija `inter, d2e, envia, xcargo, servientrega` (`Function.cs:100`), sin TCC |

## Decisión

El worker pide la etiqueta a la transportadora con su propio adaptador y la escribe **directo en S3 con el SDK**, en los mismos buckets y con el mismo formato de key y URL que `ApiLambdaPutObjectAOT/Function.cs:94-182`:

- `.txt` base64 en `bucket-guias-inter-prod` (key `{numeroGuia}.txt`).
- PDF en `bucket-guias-pdf-prod` (key `guias-pdf/{transportadora}/{guid}.pdf`) y `UrlGuia` en `PedidosInter`.

## Por qué

- Destino idéntico: Angular y la plataforma no notan diferencia.
- La máquina de estados sabe si la etiqueta quedó, reintenta y usa DLQ.
- Sin dependencia de una ruta pública sin autenticación ni de un token de usuario que el integrador M2M no tiene.

## Consecuencias

- En F3 se porta la lógica de etiqueta de cada transportadora desde `ApiLambdaCrearEtiquetaManual`, corrigiendo sus bugs.
- Rol IAM del worker limitado a esos buckets y prefijos.
- Hallazgo aparte: `PUT /putObject` y `POST /actualizaInventarioLotes` en `API_LogiGho_Rest` (`7elgyten88`) tienen `authorizationType=NONE`.
