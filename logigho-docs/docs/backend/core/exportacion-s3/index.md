## Autor: Iker Acevedo

Fecha creacion: 2026-11-24  
Estado: probado en PreProd el 2026-11-24 (3 colecciones, CSV idéntico byte a byte al del código actual de producción); producción pendiente de despliegue. Spec 037

---
## Lambda: ApiLambdaGuardaTablaS3AOT

**Trigger:** Step Functions `Step_Procesos_Logigho` (estados `Almacen S3 <Tabla>`, cada 3 horas, hora Colombia 05:00, 08:00, 11:00, 14:00, 17:00, 20:00 y 23:00)  
**AOT:** Sí, Native AOT (.NET 8, runtime `dotnet8` con ejecutable nativo como handler, x86_64). Producción: 10240 MB, timeout 900 s, dentro de VPC

---

## ¿Qué hace?

Exporta **una colección completa** de `LogighoDB` a un archivo CSV en S3 (`s3://archivos-sincronizacion/<Tabla>.csv` en producción). Es la fuente de los archivos que consumen los reportes. Cada ejecución del Step invoca la Lambda una vez por colección (`LiquidacionesLogigho`, `PedidosInter`, `Tiendas`, `Inventario`, etc.).

Pertenece al dominio **core**.

---

## Accionador

Invocación directa desde Step Functions (`lambda:invoke`) con un evento con forma de API Gateway; solo importa el campo `body`.

---

## Request (campo `body`, JSON)

```json
{
  "bucketName": "archivos-sincronizacion",
  "key": "LiquidacionesLogigho.csv",
  "textContent": "",
  "boolCompress": false,
  "nombreTabla": "LiquidacionesLogigho"
}
```

| Campo | Tipo | Requerido | Descripción |
| ----- | ---- | --------- | ----------- |
| `nombreTabla` | `string` | Sí | Colección a exportar. Solo letras, números, punto, guion y guion bajo; no puede empezar por `system.` |
| `bucketName` | `string` | No | Bucket de destino. Si falta usa el bucket por defecto de la Lambda |
| `key` | `string` | No | Nombre del archivo. Si falta usa `<nombreTabla>.csv` |
| `textContent`, `boolCompress` | | No | Se envían por compatibilidad; esta Lambda no los usa |

## Response

| Código | Cuándo | Cuerpo |
| ------ | ------ | ------ |
| `200` | CSV generado y subido | `{"Error":false,"Resultado":{"result":"Carga por lotes completada"}}` |
| `400` | Falta `nombreTabla`, JSON inválido o nombre no permitido | `{"Error":true,"Mensaje":"..."}` |
| `500` | Error de Mongo o de S3 | `{"Error":true,"Mensaje":"Excepcion no Controlada: ..."}` |

!!! warning "El Step no ve el código HTTP"
    Observado: la invocación de la Lambda termina con éxito (`StatusCode` 200, sin `FunctionError`) aunque el cuerpo traiga `statusCode` 500. Sin verificar: cómo trata esa respuesta el estado del Step. Mientras no se revise, conviene mirar el cuerpo y CloudWatch (ver Logs). Mejora M-405.

---

## Cómo funciona

1. **Encabezados globales.** Recorre **toda** la colección por `_id` ascendente en lotes de 50.000 y arma la lista de columnas. Los documentos anidados salen como `Campo.Sub` y los arreglos como `Campo[0]`, `Campo[0].sub`. `_id` va primero. Así el CSV tiene las mismas columnas en todas sus filas aunque los documentos no sean homogéneos.
2. **Filas.** Lee la colección por `_id` descendente, en lotes de 50.000, por **rango** (`_id` menor que el último leído). Mientras convierte y sube un lote, ya pide el siguiente a Mongo. Cada fila se arma con los mismos encabezados.
3. **Subida.** Sube a S3 por partes (multipart, partes de al menos 5 MB). El archivo anterior **solo se reemplaza cuando la carga termina completa**; si algo falla, el CSV anterior queda intacto.
4. **Control de integridad.** Compara las filas exportadas con el conteo estimado de la colección (`WARN` si la diferencia supera 500 o el 0,1 %).

### Formato del CSV

- Primera línea: encabezados. Una fila por documento, en orden `_id` descendente (los más nuevos primero).
- Documento anidado `cliente: {uno, dos, tres}` → columnas `cliente.uno`, `cliente.dos`, `cliente.tres`.
- Arreglo de objetos `Novedad: [{estado}, {estado}]` → columnas `Novedad[0].estado`, `Novedad[1].estado`. Si un documento tiene menos elementos, esas celdas quedan vacías.
- Un campo que no existe en un documento queda vacío. Las fechas salen como `yyyy-MM-ddTHH:mm:ssZ`. Se reemplazan por espacio los caracteres de control y las comas dentro del texto.
- Una colección vacía genera un CSV con solo la línea de encabezados (`_id`).

---

## Logs (CloudWatch)

Una línea de texto por evento, con `requestId` para seguir una ejecución. Nunca se escriben valores de documentos ni cadenas de conexión.

| Prefijo | Significado |
| ------- | ----------- |
| `[REQUEST]` | Tabla, bucket, clave y tamaño de lote |
| `[INFO] encabezados` | Cuántas columnas y cuánto tardó la pasada de encabezados |
| `[INFO] pagina N` | Filas acumuladas, columnas y segundos de la página |
| `[INFO] integridad ok` | Filas exportadas contra el total estimado |
| `[WARN]` | Solicitud inválida, total estimado distinto o no estimable |
| `[ERROR]` | Falla al calcular encabezados o al exportar (con filas y páginas hechas) |
| `[RESULT]` | Resultado final con filas, páginas, columnas y segundos |

---

## Datos y variables

- Colecciones: cualquiera de `LogighoDB` (lectura). No escribe en Mongo.
- Variables de entorno (solo nombres): `CADENA_CONEXION` (cifrada AES-256), `DATABASE_NAME`.
- Producción: `s3://archivos-sincronizacion`. PreProd: `archivos-sincronizacion-preproduccion`.

## Por qué cambió (2026-11-24)

Entre el 2026-11-22 y el 2026-11-24 el Step falló en 9 de las 10 últimas ejecuciones porque `Almacen S3 Liquidaciones` y `Almacen S3 Pedidos` agotaban los 900 s de la Lambda. La causa: la lectura usaba `Skip`, que crece con la profundidad (en producción, leer 50.000 documentos de `LiquidacionesLogigho` costaba 3,8 s al inicio, 11,9 s a mitad y 21 s al final), y además contaba la colección completa en cada página. Ver [ADR-001](ADR-001-lectura-por-rango-de-id.md).

## Pruebas

- `ApiLambdaGuardaTablaS3AOT.Tests` (61): bucle de exportación, servicio y handler. Incluye la comparación contra el algoritmo anterior (encabezados globales + `Skip`): el CSV sale **idéntico byte a byte** con datos homogéneos, heterogéneos, anidados y arreglos.
- `AccesoDatos.Tests` (8): la consulta por rango con una colección falsa estricta que falla si se usa `Count` o `Skip`.
- `Servicios.Tests` (14, ya existentes): sin cambios.

## Despliegue y reversión

Pipelines del dominio core en `Lambdas_NET_LogiGho/pipelines/core-exportacion-s3/`: `ci.yml`, `cd-preprod.yml` (se dispara en `release/*`) y `cd-prod.yml` (se dispara en `main`, con aprobación del Environment `produccion`). El CD publica solo el código: no cambia memoria, timeout, VPC ni variables. Antes de publicar guarda una copia del código anterior en `s3://logigho-artefactos-prod/respaldo/ApiLambdaGuardaTablaS3AOT/` y deja impreso el comando de reversión.

## Límites conocidos

- El tiempo crece de forma lineal con el número de documentos: la pasada de encabezados lee la colección completa y luego se vuelve a leer para las filas. Para volúmenes mayores a 2-2,5 millones de documentos hace falta otro diseño (mejora M-402).
- La Lambda sigue en .NET 8 (excepción a la regla de .NET 10 hasta migrar `Servicios` y `AccesoDatos`).
- El Step no revisa el cuerpo de la respuesta (M-405).
