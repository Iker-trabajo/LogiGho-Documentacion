---
autor: Iker Acevedo
fecha_creacion: 2026-10-22
ultima_actualizacion: 2026-10-22
estado: diseño
---

# Endpoints — API pública de guías

> Contrato preliminar. Los campos exactos de cada guía y sus validaciones salen de `reglas-negocio.md` (en construcción). El contrato final se publicará en OpenAPI con regla, ejemplo y mensaje por campo.

Base: `ApiIntegracionesExternas` (`rl7zt6zak5`), versión `/v1`.

---

## Headers comunes

| Header | Requerido | Descripción |
| --- | --- | --- |
| `Authorization` | Sí | `Bearer <access_token>` obtenido con `client_credentials` en Cognito |
| `x-api-key` | Sí | API key del cliente (límite y cuota en el gateway) |
| `Idempotency-Key` | Sí en `POST` | Valor único por guía o lote. Ver [ADR-003](adr-003-idempotencia.md) |
| `Accept-Language` | No | Idioma de mensajes de error (default `es`) |

---

## Integrador

| Método | Ruta | Scope | Descripción |
| --- | --- | --- | --- |
| `POST` | `/v1/guias` | `guias:crear` | Crea una guía. `201` si la guía queda creada dentro de la espera (20 s por defecto, RN-066); `422 TRANSPORTADORA_RECHAZO` si la rechazan en ese lapso; `202` con `solicitudId` si el worker tarda más (consultar `GET /v1/solicitudes/{id}`) |
| `GET` | `/v1/solicitudes/{id}` | `guias:leer` | Estado de una solicitud (`RECIBIDA`, `CREANDO`, `CREADA`, `FALLIDA`, `INCIERTA`, `ANULADA`) |
| `GET` | `/v1/guias/{numero}/etiqueta` | `guias:leer` | URL temporal del PDF (`formato=sticker|mediana`) |
| `POST` | `/v1/guias/{numero}/anulacion` | `guias:anular` | Anula la guía en la transportadora y libera cupo |
| `POST` | `/v1/lotes` | `guias:crear` | Varias guías en una sola llamada. Valida todo antes; `202` con `loteId` |
| `GET` | `/v1/lotes/{id}` | `guias:leer` | Progreso y resultado por guía del lote |
| `GET` | `/v1/cupo` | `cupo:leer` | Disponible, usado y límite por tienda |
| `GET` | `/v1/tiendas`, `/v1/productos`, `/v1/transportadoras` | `guias:leer` | Catálogos permitidos al cliente |

La consulta de guías y pedidos sigue en `GET /pedidos` (ya existe).

---

## Administración

Autenticación con el pool interno `us-east-1_mncKiG8Fo` (grupo de administradores). La usan el Angular y gerencia; todo cambio queda en `ApiAuditoria`.

| Método | Ruta | Descripción |
| --- | --- | --- |
| `POST`/`GET`/`PATCH` | `/admin/v1/clientes-api` | Alta, consulta y edición de clientes |
| `POST` | `/admin/v1/clientes-api/{id}/bloquear` · `/desbloquear` | Bloqueo (perfil + API key) |
| `POST` | `/admin/v1/clientes-api/{id}/rotar-secreto` | Nuevo secreto, se muestra una sola vez |
| `PUT` | `/admin/v1/clientes-api/{id}/tiendas` · `/productos` · `/transportadoras` · `/ips` | Permisos |
| `PUT` | `/admin/v1/clientes-api/{id}/cupos` | Límites y modo de cupo |
| `GET` | `/admin/v1/clientes-api/{id}/auditoria` | Todo lo que hizo el cliente y lo que se le cambió |
| `GET` | `/admin/v1/guias/{numero}/trazas` | Request/response enviados a la transportadora |
| `GET`/`POST` | `/admin/v1/cupos/{idTienda}` · `/ajuste` | Cupo de una tienda y ajustes manuales |

---

## Errores

Formato ProblemDetails (RFC 9457) con `codigo` estable y `traceId`. Nunca se devuelve stack ni mensaje interno.

| Código | HTTP | Cuándo |
| --- | --- | --- |
| `VALIDACION` | 400 | Campo inválido; incluye lista por campo |
| `NO_AUTENTICADO` | 401 | La llamada llegó sin los claims del authorizer (no debería pasar si el gateway está bien configurado) |
| `CLIENTE_NO_REGISTRADO` | 403 | Token válido de Cognito pero el cliente no tiene perfil en `ApiClientes` |
| `CLIENTE_BLOQUEADO` | 403 | Cliente bloqueado o suspendido |
| `SCOPE_INSUFICIENTE` | 403 | El token no trae el scope que exige el método (el Lambda lo revisa además del authorizer) |
| `IP_NO_PERMITIDA` | 403 | El cliente tiene lista de IPs y la de origen no está en ella |
| `ACTUA_COMO_NO_AUTORIZADO` | 403 | El cliente mandó `X-Actua-Como-Usuario` sin ser el cliente delegado del MCP (o el cliente de la persona también es un delegado) |
| `ACTUA_COMO_REQUERIDO` | 403 | Es el cliente delegado del MCP y llamó sin `X-Actua-Como-Usuario`: no opera por sí mismo |
| `USUARIO_NO_REGISTRADO` | 403 | El `sub` del header no pertenece a ningún cliente, es ambiguo (en dos clientes), viene vacío, repetido o mal formado |
| `USUARIO_BLOQUEADO` | 403 | La persona está dada de baja en `Usuarios` de su cliente |
| `TIENDA_NO_PERMITIDA` · `PRODUCTO_NO_PERMITIDO` · `TRANSPORTADORA_NO_PERMITIDA` | 403 | Fuera de los permisos del cliente |
| `CIUDAD_SIN_COBERTURA` | 422 | La transportadora no cubre el destino |
| `CUPO_INSUFICIENTE` | 422 | Detalle: disponible vs requerido |
| `IDEMPOTENCY_CONFLICTO` | 422 | Key reusada con otro body |
| `SOLICITUD_EN_CURSO` | 409 | Key en proceso |
| `TRANSPORTADORA_RECHAZO` | 422 | La transportadora rechazó. Incluye `transportadora`, código y mensaje originales y mensaje normalizado |
| `TRANSPORTADORA_NO_DISPONIBLE` | 503 | Transportadora caída o circuito abierto |
