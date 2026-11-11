# API de administración de experiencias (`/experiencias`)

**Para quién es:** el equipo de desarrollo. Con esta API se **crean, configuran, activan, desactivan y retiran** las experiencias visuales del
[Sistema de Experiencias](../frontend/experiencias/index.md) **desde un panel de la aplicación**, sin tocar la base de datos ni variables de entorno.
Es una herramienta **solo para desarrollo**; los usuarios finales solo usan [`/preferencias/mia`](preferencias.md).

!!! danger "No usar `/metodoGenerico` para estas colecciones"
    Las colecciones `Experiencias` y `PreferenciasUsuario` se escriben **únicamente** con la Lambda `ApiLambdaPreferenciasUsuario` (las rutas de esta
    página y de `/preferencias/mia`). **Nunca** las modifiquen con `/metodoGenerico` ni con `insertarGenerico`, MongoDB Compass o scripts propios (la única excepción son los scripts de preparación del repo, que solo crean índices y la semilla de `clasica`):
    se saltan las reglas de negocio (la experiencia `clasica` intocable, el control de versión, el formato de los ids, la auditoría en logs).
    La Lambda genérica hoy **no** tiene lista blanca de colecciones, así que el sistema no lo impide: **lo impide el equipo**. Un documento
    inválido se ignora al leer el catálogo (nunca llega a los usuarios), pero un cambio mal hecho puede dejar una experiencia sin aparecer.

| Entorno | URL base | Estado |
|---|---|---|
| PreProd | `https://8u5isw5u30.execute-api.us-east-1.amazonaws.com/PreProd` | Disponible cuando el backend se despliega y valida en PreProd |
| Producción | `https://7elgyten88.execute-api.us-east-1.amazonaws.com/Produccion` | **Todavía no.** Al montarla, `ROLES_ADMIN_EXPERIENCIAS` puede quedar vacío (nadie administra) |

Autenticación y errores: igual que [`/preferencias/mia`](preferencias.md#autenticacion): header `Token: <id_token de Cognito>`, errores `{ "codigo", "traceId" }`.

## Quién puede qué

| Acción | Quién |
|---|---|
| **Leer** (`GET`) | **Cualquier usuario autenticado**, pero cada uno solo ve las experiencias que sus roles le permiten (misma regla que `disponibles`). Lo que no puede ver responde `404`, igual que lo que no existe |
| **Crear, editar, cambiar estado, borrar** | Solo roles de la variable de entorno **`ROLES_ADMIN_EXPERIENCIAS`** (en PreProd: `Desarrollador,CEO`). Los roles se leen de `Users` por el `sub` del token; nunca del cuerpo. Variable vacía = nadie administra |
| Ver **todo el catálogo** (`?todas=true`) y los campos de administración | Solo administradores |

## La colección `Experiencias`

```json
{
  "id": "prueba-interna",         // único, no cambia: ^[a-z][a-z0-9-]{1,39}$
  "nombre": "Prueba interna",     // 2-60 caracteres, sin < >
  "descripcion": "Para probar",   // 0-200 caracteres
  "estado": "borrador",           // borrador | beta | general | retirada
  "rolesAcceso": ["Piloto"],      // roles EXTRA que la ven mientras esté en borrador o beta (máx. 20)
  "temas": ["claro"],             // 1-10 ids de tema
  "temaPorDefecto": "claro",      // debe estar en temas
  "orden": 100,                   // 0-9999, menor primero en el selector
  "version": 3,                   // control de concurrencia
  "creadoEn": "2026-11-10T12:00:00Z", "creadoPor": "<sub>",
  "actualizadoEn": "2026-11-10T12:00:00Z", "actualizadoPor": "<sub>"
}
```

**Quién ve cada estado:** `general` → todos · `beta` → rol `Experiencia`, `Desarrollador` y los de `rolesAcceso` · `borrador` → `Desarrollador` y los de
`rolesAcceso` · `retirada` → nadie. No hay límite de experiencias ni reglas de transición: **cualquier estado puede pasar a cualquier otro**
(activar, desactivar, volver a borrador, retirar, reactivar). La única excepción es `clasica`.

**La experiencia `clasica` es intocable:** siempre existe, siempre es `general` y no se puede borrar ni cambiar de estado (`403 EXPERIENCIA_PROTEGIDA`).
Sí se pueden editar su nombre, descripción, orden y temas. Aunque la colección esté vacía o alguien manipule su documento, la plataforma sigue
sirviendo `clasica` como `general`.

**Qué pasa con los usuarios cuando se cambia una experiencia:** nada se modifica en sus datos. Quien tenía guardada una experiencia que ya no puede ver
recibe `clasica` en su siguiente `GET /preferencias/mia`. Si el tema que tenía ya no existe en su experiencia, recibe el `temaPorDefecto`.

**Los cambios no son instantáneos:** la Lambda cachea el catálogo en memoria **60 segundos** por instancia. El cambio se ve de inmediato en la instancia que
lo hizo y en máximo ~60 s en las demás (`CACHE_CATALOGO_SEG`). Las lecturas de administración (`GET` de un administrador) van siempre a la colección.

## Endpoints

| Método y ruta | Quién | Éxito |
|---|---|---|
| `GET /experiencias` | cualquier autenticado | `200` `{ "experiencias": [...] }` |
| `GET /experiencias/{id}` | cualquier autenticado | `200` |
| `POST /experiencias` | admin | **`201`** + el documento |
| `PUT /experiencias/{id}` | admin | `200` + el documento |
| `PUT /experiencias/{id}/estado` | admin | `200` + el documento |
| `DELETE /experiencias/{id}?version=N` | admin | **`204`** sin cuerpo |

El `OPTIONS` de cada ruta lo responde la Lambda (preflight CORS, sin token).

**Vista pública** (usuario común): `id, nombre, descripcion, estado, temas, temaPorDefecto, orden`.
**Vista de administración** (admin): además `rolesAcceso, version, creadoEn, creadoPor, actualizadoEn, actualizadoPor`.

### `GET /experiencias`
Lo visible para el usuario, ordenado por `orden`. `?todas=true` (solo admin) devuelve todo el catálogo, incluidas las `retirada`. Un usuario no admin con `todas=true` recibe `403 SIN_PERMISO`. Cualquier valor de `todas` distinto de `true` o `false` es `400`.

### `POST /experiencias`
```json
{ "id": "prueba-interna", "nombre": "Prueba interna", "descripcion": "Para probar", "estado": "borrador",
  "rolesAcceso": [], "temas": ["claro"], "temaPorDefecto": "claro", "orden": 100 }
```
Solo `id` y `nombre` son obligatorios. Por defecto: `estado` `borrador`, `rolesAcceso` `[]`, `temas` `["claro"]`, `temaPorDefecto` `"claro"`, `orden` `100`. Nace con `version` 1.

### `PUT /experiencias/{id}` (editar)
```json
{ "nombre": "Prueba interna 2", "descripcion": "", "rolesAcceso": ["Piloto"], "temas": ["claro", "oscuro"], "temaPorDefecto": "claro", "orden": 120, "version": 3 }
```
Reemplaza **todos** los campos editables (excepto `descripcion`, opcional). No cambia el `id` ni el `estado`. Suma 1 a la `version`.

### `PUT /experiencias/{id}/estado`
```json
{ "estado": "general", "version": 4 }
```
Cualquier estado a cualquier otro. Para `clasica` siempre `403 EXPERIENCIA_PROTEGIDA`.

### `DELETE /experiencias/{id}?version=N`
La `version` va en la query (1 a 1 000 000). **Regla de borrado:** solo se borra si la experiencia está en `borrador` o `retirada` **y** ningún usuario la tiene
guardada. En cualquier otro caso no se borra: se **retira** (`PUT …/estado` con `retirada`). El orden en que se comprueba y lo que responde:

| # | Condición | HTTP | `codigo` | Qué hacer |
|---|---|---|---|---|
| 1 | No es administrador | 403 | `SIN_PERMISO` | — |
| 2 | Es `clasica` | 403 | `EXPERIENCIA_PROTEGIDA` | No se puede borrar nunca |
| 3 | No existe | 404 | `EXPERIENCIA_NO_ENCONTRADA` | — |
| 4 | La `version` no es la actual | 409 | `CONFLICTO_VERSION` | Volver a leerla (`GET`) y reintentar |
| 5 | Está en `beta` o `general` | 409 | `ESTADO_NO_PERMITE_BORRAR` | Pasarla antes a `borrador` o `retirada` |
| 6 | Algún usuario la tiene guardada | 409 | `EXPERIENCIA_EN_USO` | No se borra: se deja **retirada** (los usuarios reciben `clasica`) |
| — | Todo en orden | **204** | — | Borrada, sin cuerpo |

Un `version` ausente o mal formado es `400 VALIDACION`. Si alguien guarda justo esa experiencia entre la comprobación y el borrado (o desde una instancia con la caché vieja), el borrado puede
alcanzar a ese usuario: él recibirá `clasica` (mismo efecto que retirar) y queda un `[WARN] eliminada-con-usuarios` en los logs. **Si se vuelve a crear el mismo `id` con otro contenido, ese usuario lo heredaría sin haberlo elegido**: evite reutilizar ids borrados.

## Detalles que conviene saber

- **El administrador ve más:** `GET /experiencias/{id}` y `GET /experiencias?todas=true` devuelven a un administrador cualquier experiencia (incluidas `borrador` y `retirada`) para poder editarla; un usuario común solo ve lo que su rol permite.
- **Un documento inválido en la colección** (alterado a mano, por ejemplo con un estado que no existe) se **ignora** al leer el catálogo: la API lo trata como inexistente (`GET`/`PUT`/`DELETE` dan `404`) y un `POST` con ese mismo `id` da `409 EXPERIENCIA_YA_EXISTE`. Un `[WARN] experiencia-invalida id=… motivo=…` queda en los logs. Se repara corrigiendo el documento (por un script de Iker, nunca con `/metodoGenerico`).
- **Caracteres no permitidos:** el nombre, la descripción y los roles rechazan `<` `>` y los caracteres de control, de formato (invisibles, de dirección, como U+200B o U+202E), separadores de línea y párrafo, y UTF-16 inválido (`400 VALIDACION`).
- **El permiso se comprueba antes que el cuerpo:** un usuario sin rol de administración recibe `403 SIN_PERMISO` aunque su cuerpo sea inválido.

## Todos los códigos

| HTTP | `codigo` | Cuándo |
|---|---|---|
| 400 | `VALIDACION` | JSON inválido, campo extra/repetido/faltante, tipo o formato incorrecto, id de la ruta mal formado, `version` mal formada |
| 401 | `SIN_TOKEN` · `TOKEN_INVALIDO` · `USUARIO_NO_ENCONTRADO` | Sin sesión válida |
| 403 | `SIN_PERMISO` | El usuario no tiene un rol de `ROLES_ADMIN_EXPERIENCIAS` (o pidió `todas=true`) |
| 403 | `EXPERIENCIA_PROTEGIDA` | Borrar o cambiar de estado `clasica` |
| 404 | `EXPERIENCIA_NO_ENCONTRADA` | No existe **o no es visible para el usuario** |
| 405 | `METODO_NO_PERMITIDO` | Método que la ruta no admite (la cabecera `Allow` lista los permitidos) |
| 409 | `EXPERIENCIA_YA_EXISTE` | `POST` con un `id` que ya existe (incluido `clasica`) |
| 409 | `CONFLICTO_VERSION` | La `version` enviada no es la guardada |
| 409 | `ESTADO_NO_PERMITE_BORRAR` · `EXPERIENCIA_EN_USO` | Ver la tabla del `DELETE` |
| 413 | `CUERPO_DEMASIADO_GRANDE` | Más de 8192 bytes |
| 500 | `ERROR_INTERNO` | Fallo no controlado, configuración inválida o falta un índice único (sin detalles) |

## Auditoría

La auditoría va a los **logs de la Lambda** (CloudWatch `/aws/lambda/ApiLambdaPreferenciasUsuario`), una línea por escritura, sin nombres ni datos personales:

```
[AUDIT] trace=3fa85f6457a1 sub=<sub> accion=ESTADO id=prueba-interna estado=beta->general
[AUDIT] trace=…               sub=<sub> accion=CREAR|EDITAR|ESTADO|ELIMINAR id=… 
```
Además cada documento guarda `creadoPor/En` y `actualizadoPor/En`. No existe (por ahora) una colección de historial; si los controles de auditoría lo piden, se crea.

## Cómo consumirla (panel de desarrollo)

1. Al abrir el panel: `GET /experiencias?todas=true` (solo se muestra a quien tenga rol de administración; para los demás, el front oculta el panel).
2. Guarde la `version` de cada documento y envíela en cada `PUT` y `DELETE`. Ante `409 CONFLICTO_VERSION`: vuelva a leer y muestre el cambio.
3. Muestre `nombre`, `descripcion` y `estado` con `textContent` (nunca `innerHTML`): aunque la API rechaza `<` y `>`, el front no debe confiar en ello.
4. Cree la experiencia en `borrador`, pruébela con un usuario `Desarrollador` (o con `rolesAcceso`), pásela a `beta`, y por último a `general`.
5. Al crear una experiencia recuerde que **el código de la experiencia vive en el front** (`experiencias/<id>/`): si el `id` no está en el registro del front, el front la ignora y usa `clasica`.
6. Los cambios pueden tardar hasta ~60 s en verse para todos los usuarios.

## Documentos relacionados
[API de Preferencias](preferencias.md) · [OpenAPI](preferencias.openapi.yaml) · [Ficha de la Lambda y operación](../backend/core/preferencias-usuario/index.md) · [ADR-002](../backend/core/preferencias-usuario/ADR-002-catalogo-en-base-de-datos-y-aot.md)
