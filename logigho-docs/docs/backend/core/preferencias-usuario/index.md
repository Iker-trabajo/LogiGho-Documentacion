## Autor: Iker Acevedo

Fecha creacion: 2026-11-10  
Estado: desarrollo (código y pruebas listos; despliegue y validación en PreProd en curso; producción no montada). Specs 013 y 014

---
## Lambda: ApiLambdaPreferenciasUsuario

**Trigger:** API Gateway (REST `API_LogiGho_Rest`, dominio `core`)  
**AOT:** Sí, Native AOT real (.NET 10, runtime `provided.al2023`, ejecutable `bootstrap`, x86_64, 512 MB, timeout 10 s)

---

## ¿Qué hace?

Hace dos cosas. (1) **Preferencias:** guarda la experiencia visual que elige cada usuario del [Sistema de Experiencias](../../../frontend/experiencias/index.md) y le dice cuáles
puede usar. La disponibilidad (`general`, `beta`, `borrador`, `retirada`) la decide el servidor con un catálogo
configurado por variables de entorno y con los roles que el usuario tiene en `Users`.

(2) **Administración de experiencias (solo desarrollo):** CRUD del catálogo (colección `Experiencias`) para crear, configurar, activar, desactivar y retirar
experiencias desde un panel, sin tocar la base de datos. Solo esta Lambda escribe `Experiencias` y `PreferenciasUsuario`.

Contrato para el front: [API de Preferencias](../../../api/preferencias.md) · [API de administración](../../../api/experiencias.md) · [OpenAPI](../../../api/preferencias.openapi.yaml).

---

## Accionador

| Método | Ruta | Auth |
| ------ | ---- | ---- |
| `GET` | `/preferencias/mia` | Authorizer Cognito (`Token`) + JWT verificado en la Lambda |
| `PUT` | `/preferencias/mia` | Authorizer Cognito (`Token`) + JWT verificado en la Lambda |
| `GET` | `/experiencias`, `/experiencias/{id}` | Cualquier rol autenticado (ve solo lo que su rol permite) |
| `POST` | `/experiencias` | Roles de `ROLES_ADMIN_EXPERIENCIAS` |
| `PUT` | `/experiencias/{id}`, `/experiencias/{id}/estado` | Roles de `ROLES_ADMIN_EXPERIENCIAS` |
| `DELETE` | `/experiencias/{id}?version=N` | Roles de `ROLES_ADMIN_EXPERIENCIAS` |
| `OPTIONS` | todas las rutas | Ninguna (preflight CORS sin datos; lo responde la Lambda) |

REST de PreProd `8u5isw5u30` (stage `PreProd`) y de producción `7elgyten88` (stage `Produccion`).

---

## Request / Response

Ver el contrato completo en [API de Preferencias](../../../api/preferencias.md). Resumen:

```json
// PUT /preferencias/mia  (máx. 2048 bytes, exactamente estos campos)
{ "experiencia": "prueba-interna", "tema": "claro", "version": 3 }
```

| Código | Cuándo |
| ------ | ------ |
| `200` | Guardado (`version` + 1) / lectura |
| `400` `VALIDACION` | JSON inválido, campo extra o repetido, tipo o formato incorrecto |
| `401` | Sin token, token no verificable o usuario sin registro |
| `409` `CONFLICTO_VERSION` | La versión enviada no es la guardada |
| `413` | Cuerpo mayor a 2048 bytes |
| `422` | Experiencia o tema no disponibles para el usuario |
| `500` `ERROR_INTERNO` | Fallo no controlado o configuración inválida (sin detalles) |

---

## Flujo interno

```
Function (handler REST v1)
  -> EnrutadorApi            ruta, método, CORS, token (401), errores -> HTTP, logs con traceId
     -> VerificadorJwtCognito   firma RS256 + iss + exp + token_use + app client  => sub
     -> ValidadorSolicitudGuardar   tamaño en bytes, JSON estricto, campos cerrados, regex
     -> ObtenerPreferenciasUseCase / GuardarPreferenciasUseCase / ExperienciasUseCase (CRUD, solo desarrollo)
          -> CatalogoCache (60 s) -> IExperienciasRepository   MongoDB: Experiencias
          -> IUsuarioRepository      MongoDB: Users (solo lectura, proyección { roles })
          -> DisponibilidadExperiencias (Dominio, función pura)
          -> IPreferenciasRepository MongoDB: PreferenciasUsuario
```

Estructura: `LambdasLogiGho.Aplicacion/Logigho/ApiLambdaPreferenciasUsuario/` (`Aplicacion`, `Dominio`, `Infraestructura`, `Test`).

### Concurrencia y datos

- Documento: `{ _id, sub, experiencia, tema, version, actualizadoEn }`. Nada más: sin roles, correo ni nombre.
- Primer guardado (`version` 0): `InsertOne`; el índice único `ux_sub` convierte el duplicado en `409`.
- Guardados siguientes: `UpdateOne` con filtro `{ sub, version }` e `$inc` de `version`; cero coincidencias = `409`. Sin `upsert`.
- **El índice `ux_sub` es requisito**: sin él, dos primeros guardados simultáneos podrían crear dos documentos. La Lambda lo verifica sola antes de escribir.
- Compatible con DocumentDB (`retryWrites=false`, sin transacciones ni operadores especiales).

---

## Configuración (variables de entorno, las gestiona Terraform)

| Variable | Qué es |
|---|---|
| `ROLES_ADMIN_EXPERIENCIAS` | Roles de `Users` que pueden crear, editar, cambiar de estado y borrar experiencias. Vacío o ausente = nadie (la lectura siempre es abierta). PreProd: `Desarrollador,CEO` |
| `COLECCION_EXPERIENCIAS`, `CACHE_CATALOGO_SEG` | Colección del catálogo (`Experiencias`) y segundos de caché en memoria (0-3600, por defecto 60) |
| `ROLES_BORRADOR` | Roles de `Users` que ven `borrador` (y `beta`). Por defecto `Desarrollador` |
| `ROLES_BETA` | Roles que ven `beta`. Por defecto `Experiencia` |
| `ORIGEN_PERMITIDO` | Orígenes CORS exactos, separados por coma (nunca `*`) |
| `COGNITO_REGION`, `COGNITO_USER_POOL_ID`, `COGNITO_CLIENT_IDS` | Verificación del JWT |
| `CADENA_CONEXION`, `DATABASE_NAME` | Base de datos (cadena cifrada AES, como las demás Lambdas) |
| `COLECCION_PREFERENCIAS`, `COLECCION_USUARIOS`, `MAX_TIME_MS` | Nombres de colecciones y tiempo máximo por consulta (3000 ms) |

**Falla cerrada:** si los roles, los orígenes, Cognito, la caché o la conexión son inválidos, la Lambda responde `500 ERROR_INTERNO` a todo y escribe un `[ERROR]` con la
causa (sin valores). No sirve nunca con una configuración rota.
Esto incluye `COGNITO_*` ausentes o con formato inválido (un 401 a todos se confundiría con una sesión vencida). Además, antes del primer `INSERT` la Lambda comprueba que exista el índice único `ux_sub`: si falta, registra `[ERROR]` y responde 500 en el `PUT` (las lecturas siguen funcionando).

---

## Seguridad (B1–B10, [seguridad por diseño](../../../frontend/experiencias/seguridad-por-diseno.md))

| Riesgo | Control implementado |
|---|---|
| Suplantación | Authorizer Cognito en las rutas + verificación de firma del JWT en la Lambda; `sub` solo del token (formato validado) |
| Manipulación | JSON estricto (campos cerrados, sin duplicados, profundidad 4), ids con regex, disponibilidad y tema validados contra el catálogo del servidor, control optimista por `version` |
| Repudio | `actualizadoEn` en cada documento; logs con `trace`, método, ruta, `status`, duración y `sub` |
| Fuga de información | Cada usuario solo toca su documento (la ruta no admite identificadores); no se guardan ni devuelven roles; errores sin detalles |
| Denegación de servicio | Cuerpo ≤ 2048 bytes (8192 en administración), token ≤ 4096 caracteres, throttling por método (lecturas y OPTIONS 50 rps / ráfaga 100; PUT de preferencias 20 / 40; administración 10 / 20), timeout 10 s, `maxTimeMS` y tiempos de conexión cortos |
| Escalamiento de privilegios | `borrador` y `beta` los decide el servidor por rol; el front nunca envía roles |
| Infraestructura | Rol IAM propio (solo sus logs), tres permisos de invocación (solo `/preferencias/mia`, `/experiencias` y `/experiencias/*`), CORS por lista cerrada, nada creado a mano |

Logs: líneas de texto `[REQUEST]`, `[RESULT]`, `[WARN]`, `[ERROR]` con `trace=`. **Nunca** token, cabeceras, cuerpo ni mensajes de excepción (solo el tipo).

---

## Infraestructura

Terraform: módulo `modules/core-preferencias-usuario` (rol, log group, Lambda, rutas, permiso, alarmas) y stacks `environments/preprod/core` y
`environments/prod/core`. El deployment del stage de PreProd lo publica `environments/preprod/datos` (suma los `ids_para_despliegue` de `core`);
en producción lo publica Iker con el CLI.

Pipelines (Azure DevOps): `Lambdas-Core-CI`, `Lambdas-Core-CD-PreProd`, `Lambdas-Core-CD-Prod` (`pipelines/core/`). El CD publica solo código.

### Alarmas (CloudWatch → SNS)

| Alarma | Cuándo |
|---|---|
| `Preferencias-<entorno>-errores-internos` | Cualquier `[ERROR]` en 5 min (500 sin excepción) |
| `Preferencias-<entorno>-errores-lambda` | Errores de invocación en 5 min |
| `Preferencias-<entorno>-throttles` | Throttling de la Lambda |
| `Preferencias-<entorno>-duracion-p95` | p95 de Duration > 1 s durante 10 min |

---

## Operación (runbook)

| Tarea | Cómo |
|---|---|
| Crear el índice (una vez por entorno) | `python tools/indices-prod/crear_indices_preferencias.py --entorno preprod` (ensayo) y `--aplicar`; o `mongosh` con `scripts/mongo/core/001-preferencias-usuario.js`. Lo ejecuta Iker |
| Publicar una experiencia nueva en `borrador` | `POST /experiencias` (panel de desarrollo). Solo la ve `Desarrollador` y los roles de `rolesAcceso` |
| Pasar a `beta` | `PUT /experiencias/{id}/estado` con `beta`. La ven los usuarios con rol `Experiencia`, `Desarrollador` y `rolesAcceso` |
| Dar el acceso beta a una persona | Agregar el rol `Experiencia` a su `Users.roles` (lo asigna Iker/Comercial según el mapa de roles) |
| Pasar a general | `PUT …/estado` con `general`: la ven todos los usuarios (cada uno con sus módulos de siempre) |
| Retirar o desactivar | `PUT …/estado` con `retirada` (o `borrador`). Quienes la tenían guardada reciben `clasica` en el siguiente `GET`; sus datos no se tocan |
| Borrar una experiencia | `DELETE /experiencias/{id}?version=N`: solo en `borrador`/`retirada` y sin usuarios que la tengan guardada (409 `ESTADO_NO_PERMITE_BORRAR` / `EXPERIENCIA_EN_USO`). Si no, se retira |
| Revertir de inmediato | Pasar la experiencia a `retirada` (o `borrador`): todos vuelven a `clasica` sin tocar datos de usuarios, en ≤ 60 s |
| Preparar una base nueva | `python tools/indices-prod/crear_indices_preferencias.py --entorno <preprod/prod> --aplicar` (índices únicos `ux_sub` y `ux_id`, `ix_experiencia` y semilla de `clasica`) o `scripts/mongo/core/001…` y `002…` con `mongosh` |
| Auditoría | Logs de la Lambda: líneas `[AUDIT] trace=… sub=… accion=… id=…`. Cada documento guarda también `creadoPor/En` y `actualizadoPor/En` |
| Ver errores | CloudWatch Logs `/aws/lambda/ApiLambdaPreferenciasUsuario`, buscar `[ERROR]` o el `trace=` del `traceId` que reportó el front |
| Apagar la API sin afectar a nadie | El front usa `clasica` si la API falla; basta desactivar la Lambda o las rutas |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-11-10 | Iker Acevedo | Primera versión: Lambda, colección, Terraform, pipelines y documentación (spec 013) |
| 2026-11-10 | Iker Acevedo | Catálogo en la colección `Experiencias`, CRUD de administración y migración a Native AOT (spec 014) |

---

## Observaciones

- La Lambda **verifica el JWT ella misma** además del authorizer (decisión de [ADR-001](ADR-001-identidad-y-catalogo-del-servidor.md)). El código del verificador está copiado del de métricas e inventario: extraerlo a una librería común es una mejora pendiente.
- Las variables de entorno son de Terraform (no del pipeline). El **catálogo de experiencias ya no es una variable**: vive en la colección `Experiencias` y se administra por la API ([ADR-002](ADR-002-catalogo-en-base-de-datos-y-aot.md)).
- **No escribir `Experiencias` ni `PreferenciasUsuario` con `/metodoGenerico`, `insertarGenerico` ni Compass:** se saltan las reglas (clasica intocable, versión, ids). La Lambda genérica no tiene lista blanca (mejora en el backlog); lo impide el equipo.
- **Native AOT:** se compila en una imagen de Amazon Linux 2023 con Docker (`pipelines/core/aot/build-aot.sh`); el código no usa reflexión (JSON generado en compilación, solo `BsonDocument`). Las demás Lambdas llamadas "AOT" del repo corren hoy en `dotnet8` gestionado (observado 2026-11-10); esta es la primera con `provided.al2023`.
- **Cadena de conexión:** la lectura "fresca" de administración y la comprobación de uso al borrar deben ir al nodo primario. Si `CADENA_CONEXION` (DocumentDB) llevara `readPreference=secondary*`, podrían ir retrasadas: verificarlo al montar producción. Tampoco debe llevar `compressors=` (dependencias con CVE, M-290).
- El catálogo se cachea 60 s por instancia: un cambio de administración se ve de inmediato en la instancia que lo hizo y en ≤ 60 s en las demás.
- Un `401` del authorizer del gateway no lleva cabeceras CORS (comportamiento del REST compartido); el front lo trata como cualquier fallo.
- Un usuario con varios `cognitoId` (otro pool o entorno) tiene una preferencia por cada `sub`.
- Pendiente (mejora): usuario de base de datos con permisos mínimos (solo `PreferenciasUsuario` y lectura de `Users`).

---

## Autor: Iker Acevedo
Fecha creacion: 2026-11-10  
Estado: desarrollo
