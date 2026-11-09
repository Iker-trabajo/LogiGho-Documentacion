# Variables de entorno

!!! warning "Importante"
    Nunca subas valores reales al repositorio. Usa `.env.example` como referencia.

## Backend .NET

| Variable | Descripción | Ejemplo |
|---|---|---|
| — | — | — |

## Lambdas Python

| Variable | Descripción | Ejemplo |
|---|---|---|
| — | — | — |

## Frontend Angular

| Variable | Descripción | Archivo |
|---|---|---|
| apiUrl | URL del API Gateway | environments/environment.ts |

## Novedades Inter: Distancias y Reportes (historia 027)

Solo **nombres**: los valores viven cifrados con AES en el variable group de Azure DevOps (`novedades-inter-<entorno>-secretos`) y nunca en git ni en esta
documentación. Fuente: `pipelines/novedades/cd-preprod.yml`, `cd-prod.yml` y los `aws-lambda-*.json`. Detalle por función:
[Distancias](../backend/lambdas-dotnet/aplicacion/Interrapidisimo/ApiLambdaNovedadesInter/ApiLambdaNovedadesInterDistancias.md) ·
[Reportes](../backend/lambdas-dotnet/aplicacion/Interrapidisimo/ApiLambdaNovedadesInter/ApiLambdaNovedadesInterReportes.md) ·
[Runbook](../backend/lambdas-dotnet/aplicacion/Interrapidisimo/ApiLambdaNovedadesInter/infraestructura/runbook.md#d-historia-027-distancias-y-reportes).

| Variable | Función | ¿Cifrada? | Descripción |
|---|---|---|---|
| `CADENA_CONEXION` | Las 3 | Sí (AES) | Cadena de conexión a Mongo (Atlas en PreProd, DocumentDB en prod) |
| `DATABASE_NAME` | Distancias, Reportes | No (Terraform) | Base de datos |
| `ENTORNO` | Distancias, Reportes | No (Terraform) | `preprod` o `prod` |
| `GOOGLE_MAPS_API_KEY` | Distancias | Sí (AES) | Llave de **servidor** de Google Maps Platform (Geocoding y Routes) |
| `MAX_GUIAS_DISTANCIA_POR_CORRIDA` | Distancias | No (opcional) | Tope de guías por corrida; 1500 por defecto |
| `CONCURRENCIA_GOOGLE` | Distancias | No (opcional) | Guías en paralelo; 4 por defecto |
| `FALLAS_SEGUIDAS_PARA_CORTAR` | Distancias | No (opcional) | Fallas seguidas de Google para cortar la corrida; 5 por defecto |
| `SLACK_WEBHOOK_ALERTAS` | Distancias (prod) y función original | Sí (AES) | Webhook de alertas; opcional |
| `SLACK_BOT_TOKEN` | Reportes | Sí (AES) | Token del bot de la App «LogiGho» |
| `SLACK_SIGNING_SECRET` | Reportes | Sí (AES) | Secreto que valida la firma del comando `/reporte-novedades` |
| `SLACK_CANAL_REPORTES` | Reportes | No | ID del canal (`C…`) donde se suben los Excel; uno por entorno |
| `SLACK_USUARIOS_PERMITIDOS` | Reportes | No | IDs de usuario de Slack que pueden usar el comando, separados por `;` o `,` |
| `MOTIVOS_EXCLUIDOS_REPORTE2` | Reportes | No (opcional) | Reemplaza la lista de 8 motivos excluidos del reporte 2 (separada por `;`) |
| `ZONA_HORARIA` | Reportes | No (opcional) | `America/Bogota` por defecto |
| `EVIDENCIAS_SECRETO_FIRMA` | Reportes | Sí (**AES256**) | v8. Clave HMAC-SHA256 de los enlaces fijos de imágenes. Sin ella los Excel salen sin columnas de imagen. Rotarla invalida los enlaces ya enviados ([runbook D9](../backend/lambdas-dotnet/aplicacion/Interrapidisimo/ApiLambdaNovedadesInter/infraestructura/runbook.md#d9-rotar-evidencias_secreto_firma)) |
| `URL_EVIDENCIAS` | Reportes | No | v8. URL https base de [`GET /novedades/evidencia`](../backend/lambdas-dotnet/aplicacion/Interrapidisimo/ApiLambdaNovedadesInter/infraestructura/evidencia-endpoint.md), sin query (salida `url_evidencias` de Terraform) |
| `EVIDENCIAS_BUCKET` | Reportes | No | v8. Bucket privado de las fotos (`wsinter-evidencias-prod` en prod) |
| `EVIDENCIAS_REGION` | Reportes | No (opcional) | v8. Región del bucket para la presignación; por defecto `AWS_REGION` |

Frontend (llave en `preprod` desde los PR 628 y 637; actualizado 2026-11-09):

| Variable | Descripción | Archivo |
|---|---|---|
| `googleMapsEmbedKey` | Llave de **navegador** de Google Maps, restringida por dominio y por API (pública por diseño). La vista la usa para cargar la Maps JavaScript API (mapa propio) y, solo en intentos anteriores al último, la geocodificación inversa del navegador. Vacía = la vista usa los iframes de siempre. **No se pide ninguna ruta a Google desde el navegador** (decisión 2026-11-08) | `environments/environment.ts` y `environment.preprod.ts` |

Para cambiar una variable ya publicada: `quitar-variables` y volver a correr el CD (las variables del pipeline son aditivas).
