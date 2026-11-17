---

## Autor: Iker Acevedo  
Fecha creacion: 2026-11-09  
Estado: produccion (desplegada y verificada 2026-11-08)

# Endpoint: GET /novedades/evidencia

**Lambda / Servicio:** [ApiLambdaNovedadesInterReportes](../ApiLambdaNovedadesInterReportes.md) (misma función que genera los Excel)  
**Auth:** No requerida (sin authorizer ni API key); la autenticidad la da la firma HMAC del enlace  
**Rol:** todos (quien tenga un enlace firmado)  
**Historia:** 027 · **Spec:** 012 §5.7, RN-16, RN-17 · **Decisión:** [ADR-005](../adr/ADR-005-enlaces-fijos-firmados-para-las-imagenes-de-novedades.md)

> Marcas: **observado** (ruta citada), **Iker** (fecha) o **hipótesis**.

---

## ¿Qué hace?

Entrega las fotos de gestión app de una novedad que el robot de Inter guarda en un bucket **privado**. El Excel de los reportes lleva un enlace fijo y firmado; este endpoint comprueba la firma
y redirige (302) a una URL prefirmada de S3 que vive 300 s. Así las fotos se abren desde el Excel sin abrir el bucket (ver el ADR para las alternativas y los costos).

---

## Request

`GET {URL_EVIDENCIAS}?k={clave}&f={firma}` (API Gateway REST, integración `AWS_PROXY`, payload 1.0).

| Parámetro | Tipo | Requerido | Descripción |
| --------- | ---- | --------- | ----------- |
| `k` | query | Sí (lo exige la lambda, no API Gateway) | `s3_key` de la imagen en base64url (RFC 4648 §5) sin relleno `=` |
| `f` | query | Sí | HMAC-SHA256 en hexadecimal minúscula; clave = `EVIDENCIAS_SECRETO_FIRMA`, mensaje = el `s3_key` en UTF-8 |

Los dos parámetros son opcionales **a propósito** en API Gateway (`request_parameters = false`): si faltan, responde la lambda (400) y no API Gateway, que revelaría el contrato.

Ejemplo ficticio: `https://api.ejemplo.invalid/novedades/evidencia?k=Z2VzdGlvbi1hcHAvMTIzNDU2Nzg5MDEyLzIwMjYtMTEtMDEvMTIzNDU2Nzg5MDEyLTEuanBn&f=<64 caracteres hexadecimales>`.

Validación de la clave (`FirmaEvidencia.ClaveValida`, observado): empieza por `gestion-app/`, máximo 300 caracteres, sin `..`, sin contrabarra ni caracteres de control, extensión `.jpg`, `.jpeg` o `.png` (sin distinguir mayúsculas).

---

## Response

| Código | Cuándo | Cuerpo | Resultado en el log |
| ------ | ------ | ------ | ------------------- |
| `302` | Firma válida: `Location` = URL prefirmada de S3 `GetObject` de **300 s**, `ResponseContentDisposition=inline` y `ResponseContentType` según la extensión (`image/png` o `image/jpeg`) | vacío | `ok` |
| `400` | Falta `k` o `f` | `{"error":"Solicitud inválida."}` | `parametros_ausentes` |
| `403` | `k` no decodifica o la clave no es válida; o la firma no coincide (comparación en tiempo constante, `CryptographicOperations.FixedTimeEquals`) | `{"error":"Prohibido."}` (idéntico en ambos casos, sin pistas) | `clave_invalida`, `firma_invalida` |
| `405` | Método distinto de `GET` | `{"error":"Método no permitido."}` | `metodo-no-permitido` |
| `500` | Sin `EVIDENCIAS_BUCKET` o falló la presignación | `{"error":"Error interno."}` | `sin_configuracion`, `error_presignado` |

**Todas** las respuestas llevan `Cache-Control: no-store`. Observado: `ReportesHandler.cs`, `ResolverEvidenciaUseCase.cs`, `PresignadorS3Evidencias.cs`.

Orden de las defensas: método → parámetros presentes → `k` válida → firma. Nunca se loguea el secreto, la firma, la clave ni la URL prefirmada (solo el resultado de la tabla, y del error solo el tipo de excepción).

---

## Infraestructura

| Pieza | Detalle (observado en Terraform) |
| ----- | -------------------------------- |
| Módulo | `modules/novedades-evidencias`: recursos `/novedades` y `/evidencia`, método `GET` con `authorization = NONE` y `api_key_required = false`, integración `AWS_PROXY` hacia la función de Reportes y permiso `InvocacionEvidenciasFirmadas` restringido a `GET/novedades/evidencia` |
| Dónde se usa | Prod: `environments/prod/novedades` (REST `7elgyten88`, recurso raíz `cyyv45wowa`, stage `Produccion`). PreProd: `environments/preprod/datos` |
| Deployment | **Fuera de Terraform en prod**: el módulo no toca el API ni el stage; Iker publica con `aws apigateway create-deployment --rest-api-id 7elgyten88 --stage-name Produccion` |
| Límite de la ruta | **burst 50, rate 20** con `aws apigateway update-stage` sobre `/~1novedades~1evidencia/GET/throttling/...`; solo después del deployment (el método debe existir) |
| Permiso S3 | Política `LeerEvidenciasFirmadas` del rol de Reportes: **solo** `s3:GetObject` sobre `arn:aws:s3:::<bucket>/gestion-app/*` (sin comodines sobre otros buckets); depende de `evidencias_bucket` |
| Sin `depends_on` | Los módulos de la ruta del comando de Slack y de evidencias **no usan `depends_on`** sobre el módulo de funciones: con él, un recurso nuevo dejaba valores «desconocidos» y Terraform proponía **reemplazar la integración y el permiso del comando de Slack en vivo** (comentario en `environments/prod/novedades/main.tf`; mismo criterio que el webhook, PR 626) |

URL de la ruta: salida `url_evidencias` del entorno (va al variable group como `URL_EVIDENCIAS`).

---

## Variables (solo nombres)

`EVIDENCIAS_SECRETO_FIRMA` (cifrada AES256), `URL_EVIDENCIAS` (https, sin query), `EVIDENCIAS_BUCKET` (`wsinter-evidencias-prod` en prod) y `EVIDENCIAS_REGION` (opcional; por defecto `AWS_REGION`). Detalle en
[Variables de entorno](../../../../../../guias/variables-entorno.md). Sin secreto o sin `URL_EVIDENCIAS` el reporte sale sin columnas de imagen.

---

## Operación

- **Rotar el secreto** invalida todos los enlaces ya enviados: [runbook D9](runbook.md#d9-rotar-evidencias_secreto_firma).
- **Verificado en prod 2026-11-08 (observado, ficha del dominio):** firma válida 302 a S3 con firma válida; firma alterada 403; sin firma 400.
- **PreProd** no puede leer las fotos del bucket de prod (otra cuenta): mejora M-275.
- Quien tenga el enlace puede reenviarlo (la firma prueba autenticidad, no identidad; aceptado, ADR-005).

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-11-09 | Iker Acevedo (redactado con IA) | Creación (Lambdas PR 635, Terraform PR 636; desplegada 2026-11-08) |
