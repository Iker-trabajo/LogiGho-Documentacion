---

## Autor: Iker Acevedo  
Fecha creacion: 2026-11-09  
Estado: aceptada

# ADR-005 — Enlaces fijos con firma HMAC-SHA256 y bucket privado para las imágenes de los reportes

**Autor:** Iker Acevedo  
**Fecha:** 2026-11-09 (la decisión es de Iker 2026-11-07; verificada en producción 2026-11-08)  
**Estado:** Aceptada

> Marcas: **observado** (ruta o fuente citada), **Iker** (fecha) o **hipótesis**.

---

## Contexto

Los Excel de la gerencia (reportes 1 y 2) deben mostrar la **imagen de cada novedad** que el robot de Inter deja en el bucket
`wsinter-evidencias-prod` (campo `UltimaRespuesta.gestion_app[].s3_key`, claves `gestion-app/<guía>/<fecha>/<guía>-<n>.jpg`;
observado en prod 2026-11-07: 7 286 seguimientos con imágenes, hasta 6 por guía; spec 012 RN-16).

Restricciones:

- El archivo **se descarga y se reenvía**: el enlace tiene que abrir días o semanas después de generar el Excel, sin que nadie lo regenere.
- Las fotos son de entregas reales (puertas, fachadas, a veces personas): el bucket **no puede quedar público**.
- El bucket de producción vive en otra cuenta de AWS que PreProd (en PreProd las fotos no se pueden leer sin política cruzada; mejora **M-275**).
- Pocas lambdas: reutilizar la función de Reportes (Iker, regla del repo).

## Opciones consideradas

### Opción A — Abrir el bucket (lectura pública de `gestion-app/*`)

El Excel lleva la URL directa de S3.

**Pros:** el enlace nunca vence y no hay código ni costo de cómputo.  
**Contras:** cualquiera que conozca o adivine una clave ve la foto (las claves siguen un patrón: guía + fecha + número); no se puede revocar un enlace sin cerrar el bucket entero
(y romper todos los Excel); expone datos de entregas fuera del equipo; cambia la política de un bucket que no es de este dominio.

### Opción B — URL prefirmada de S3 de larga vigencia en el Excel

El reporte presigna cada foto al generarlo.

**Pros:** el bucket sigue privado; no hay endpoint nuevo.  
**Contras (hipótesis, según la documentación pública de AWS; no se probó en la cuenta):** una URL prefirmada vence como máximo a los 7 días y, si se firma con credenciales temporales de un
rol (el caso de una Lambda), vence antes, cuando caduca esa sesión. Un Excel reenviado la semana siguiente quedaría con todas las imágenes rotas. Además la URL de S3 viaja completa
en el archivo: quien la copia la usa hasta que venza y no hay forma de invalidarla antes.

### Opción C — Enlace fijo firmado con HMAC que redirige a una URL prefirmada de 300 s

El Excel lleva `{URL_EVIDENCIAS}?k={clave en base64url}&f={firma}`. La firma es HMAC-SHA256 en hexadecimal minúscula, con la clave `EVIDENCIAS_SECRETO_FIRMA` y como mensaje el `s3_key`
en UTF-8. El endpoint `GET /novedades/evidencia` ([página de la ruta](../infraestructura/evidencia-endpoint.md)), servido por la **misma función Reportes**, comprueba la firma y responde
`302` a una URL prefirmada de S3 de **300 s**.

**Pros:** el enlace del Excel **no vence**; el bucket **sigue privado** y ninguna URL de S3 vive más de 5 minutos; una sola función y una sola ruta; la validación es una
función pura (`FirmaEvidencia.cs`) con el mismo `Firmar` para generar y para validar, probada de ida y vuelta.  
**Contras:** cada vista cuesta una petición de API Gateway, una invocación de Lambda y un `GET` de S3; la firma prueba **autenticidad, no identidad** (quien tenga el enlace puede reenviarlo; aceptado);
**rotar el secreto invalida todos los enlaces ya enviados** (mejora **M-274**); el rol de Reportes necesita `s3:GetObject` en el bucket de prod.

## Costos

| Opción | Costo por 1 000 vistas | Costo fijo | Riesgo principal |
| ------ | ---------------------- | ---------- | ---------------- |
| A | ~0 (solo transferencia de S3) | 0 | Exposición de fotos; no revocable |
| B | ~0 (solo S3) | 0 | Excel con imágenes rotas a los pocos días (hipótesis) |
| C | **del orden de 1 centavo de dólar o menos** (hipótesis de la spec con tarifas públicas de us-east-1: ~USD 3.50 por millón de peticiones de API Gateway REST + una invocación de ~100 ms con 1 GB + un `GET` de S3 de ~USD 0.0004 por mil; **a medir en la cuenta**) | 0 | Rotar el secreto rompe los enlaces viejos |

Protección contra abuso de la ruta (observado en `environments/prod/novedades/main.tf`): límite propio del método `GET /novedades/evidencia`, **burst 50 y rate 20**,
aplicado con `update-stage` después del deployment.

## Decisión

**Se eligió:** Opción C (Iker 2026-11-07, en `docs/architecture/decisiones.md`).

**Razón:** es la única que cumple a la vez «el enlace del Excel no vence» y «las fotos siguen privadas»; el costo adicional es de centavos por cada mil vistas y el riesgo (rotar el secreto)
es controlable con un aviso previo y el procedimiento de [rotación](../infraestructura/runbook.md#d9-rotar-evidencias_secreto_firma).

**Verificado en producción 2026-11-08 (observado, workspace `docs/architecture/dominios/novedades.md`):** el Excel trae los enlaces; con firma válida la ruta responde 302 a S3 con firma válida, con firma alterada 403 y sin firma 400.

## Consecuencias

**Positivas:** imágenes visibles desde el Excel sin abrir el bucket; revocar todo es rotar un secreto; la ruta no necesita authorizer ni API key; los logs solo registran el resultado
(`ok`, `firma_invalida`, `clave_invalida`, `parametros_ausentes`), nunca el secreto, la firma, la clave ni la URL prefirmada.  
**Negativas:** una ruta pública más en el REST de producción; si el secreto no está o `URL_EVIDENCIAS` falta, el reporte sale **sin** columnas de imagen (se registra la advertencia `evidencias-deshabilitadas`);
PreProd no puede probar el flujo completo (otra cuenta); rotar invalida los Excel viejos hasta que se regeneren.

## Impacto en el código

| Módulo / Repo | Cambio |
| ------------- | ------ |
| `Lambdas_NET_LogiGho` | `FirmaEvidencia.cs` (firma y validación), `ResolverEvidenciaUseCase.cs`, `PresignadorS3Evidencias.cs`, ruta en `ReportesHandler.cs`, columnas «Imagen novedad N» en `GeneradorExcelReportes.cs` (PR 635) |
| `Terraform_Infraestructura_Logigho` | Módulo `novedades-evidencias` (ruta, método, integración y permiso) y política `s3:GetObject` solo sobre `gestion-app/*` del rol de Reportes (PR 636) |
| Azure DevOps | Variables `EVIDENCIAS_SECRETO_FIRMA` (cifrada AES256), `URL_EVIDENCIAS` y `EVIDENCIAS_BUCKET` en el variable group; el CD las lista en `claves` |

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-11-09 | Iker Acevedo (redactado con IA) | Creación a partir de la decisión del 2026-11-07 y la verificación del 2026-11-08 |

## Referencias

- [Ruta GET /novedades/evidencia](../infraestructura/evidencia-endpoint.md) · [Reportes](../ApiLambdaNovedadesInterReportes.md) · [ADR-006](ADR-006-direccion-del-mensajero-guardada-por-geocodificacion-inversa.md)
- Spec 012 RN-16, RN-17, §5.7 y §8; decisiones 2026-11-07 del workspace (`docs/architecture/decisiones.md`); mejoras M-274 y M-275 del `backlog.md`.
