# Preferencias de usuario (Sistema de Experiencias, backend) — noviembre 2026

Estado: **código, Terraform, pipelines y documentación listos; despliegue y validación en PreProd en curso. Producción no montada.**

## [2026-11-10] — Backend de preferencias para el Sistema de Experiencias

### Nuevas funcionalidades
- Lambda `ApiLambdaPreferenciasUsuario` (.NET 10): `GET` y `PUT /preferencias/mia`. Guarda la experiencia y el tema de cada usuario y devuelve las experiencias que puede usar.
- Estados de experiencia decididos por el servidor: `general` (todos), `beta` (rol `Experiencia`), `borrador` (rol `Desarrollador`), `retirada` (nadie).
- Colección `PreferenciasUsuario` con índice único por `sub`; control de concurrencia por `version`.
- Dominio `core` en Terraform y pipelines (`Lambdas-Core-CI`, `Lambdas-Core-CD-PreProd`, `Lambdas-Core-CD-Prod`) con alarmas de CloudWatch.

### Spec 014 — Administración de experiencias y Native AOT (2026-11-10)
- Catálogo en la colección `Experiencias`, administrable con `/experiencias` (crear, editar, cambiar estado, borrar) solo para desarrollo; roles por `ROLES_ADMIN_EXPERIENCIAS`. `clasica` intocable; sin reglas de transición de estado.
- `disponibles` de `GET /preferencias/mia` trae ahora `nombre`, `orden`, `temas` y `temaPorDefecto`.
- Lambda compilada como Native AOT real (`provided.al2023`).
- **No** escribir `Experiencias` ni `PreferenciasUsuario` por `/metodoGenerico`.

### Cambios
- Ninguno en lo que ven hoy los usuarios: sin la API o con cualquier fallo, el front usa la experiencia `clasica`.
- `environments/preprod/datos` suma las rutas de `core` al trigger de su deployment y crea 12 límites de peticiones (`method settings`) para las rutas nuevas.

### Documentación
- [API de administración](../api/experiencias.md), [ADR-002](../backend/core/preferencias-usuario/ADR-002-catalogo-en-base-de-datos-y-aot.md) y [Contrato de la API](../api/preferencias.md) y [OpenAPI](../api/preferencias.openapi.yaml) para el equipo de frontend.
- [Ficha de la Lambda y operación](../backend/core/preferencias-usuario/index.md) y [ADR-001](../backend/core/preferencias-usuario/ADR-001-identidad-y-catalogo-del-servidor.md).
