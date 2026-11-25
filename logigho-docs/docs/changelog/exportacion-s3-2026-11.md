# Exportación de colecciones a S3 (nov 2026)

## [2026-11-24] — La exportación a S3 deja de agotar los 900 s

Estado: probado en PreProd (2026-11-24); pendiente de despliegue en producción.

### Correcciones
- `ApiLambdaGuardaTablaS3AOT`: lee las colecciones por rango de `_id` en vez de `Skip` y sin contar la colección en cada página. Corrige el fallo de `Almacen S3 Liquidaciones` y `Almacen S3 Pedidos` (9 de las últimas 10 ejecuciones de `Step_Procesos_Logigho` fallaban por timeout).
- Una solicitud sin `nombreTabla` o con un nombre inválido responde 400 con un mensaje claro.

### Cambios
- Logs de texto con `[REQUEST]`, `[INFO]`, `[WARN]`, `[ERROR]`, `[RESULT]` y `requestId`.
- Control de integridad: se compara lo exportado con el total estimado de la colección.
- El formato del CSV **no cambia**.

### Documentación
- [Ficha de la Lambda](../backend/core/exportacion-s3/index.md) y [ADR-001](../backend/core/exportacion-s3/ADR-001-lectura-por-rango-de-id.md).
- Pipelines CI/CD del dominio core en `Lambdas_NET_LogiGho/pipelines/core-exportacion-s3/`.
