# Autenticación

Contrato de las rutas de cuentas de LogiGho. Dominio dueño: `autenticacion` (código en `Lambdas_NET_LogiGho/LambdasLogiGho.Aplicacion/AWS/Autenticacion/`). Estado: **cambios listos en código; el despliegue y la prueba punta a punta en PreProd van antes de producción**. Lo que aún no existe está marcado como *planeado*.

## Cómo se autentica una persona

1. El front inicia sesión directo contra **Cognito** (SRP) con el user pool de LogiGho. No hay una ruta de login en el API.
2. Cognito devuelve un `id_token` (JWT RS256). El front lo guarda en `sessionStorage` (`id_token`).
3. Cada llamada del usuario al API manda ese token en el header **`Token`** (no en `Authorization`).
4. La lambda verifica el token (firma con las llaves públicas del pool, emisor, `token_use`, vigencia y que el `client_id` sea uno de los autorizados). Un token falso o vencido responde `401`.
5. El **rol** del usuario nunca sale del token ni del cuerpo: la lambda lo lee de la colección `Users`, buscando por `cognitoId` (el `sub` del token).

!!! warning "`headersecurity` no es autenticación"
    El header `headersecurity` viaja dentro del bundle de Angular, así que cualquiera lo conoce. Las rutas de este dominio **lo ignoran**. Sigue viajando en otras rutas mientras se migran una a una (backlog M-193).

## Formato de respuesta de error

Las lambdas del dominio responden JSON con esta forma:

```json
{ "Error": true, "Codigo": "PROHIBIDO", "Mensaje": "No tienes permiso para crear usuarios.", "Motivos": null }
```

- `Codigo` es estable y sirve para decidir qué mostrar. `Mensaje` es texto para la persona.
- `Motivos` solo trae datos útiles y seguros (motivos de la política de contraseña, roles rechazados, campos rechazados). Nunca trae la contraseña ni datos internos.
- Un error inesperado responde `500 ERROR_INTERNO` sin detalle. El detalle queda en los logs del servidor, también sin datos personales.

## Política de contraseña

La misma que exige el user pool: **8 a 256 caracteres**, al menos una minúscula, una mayúscula, un número y un símbolo, sin espacios al inicio ni al final y sin caracteres de control. Símbolos válidos: `^ $ * . [ ] { } ( ) ? " ! @ # % & / \ , > < ' : ; | _ ~ ` = + -`.

Cuando falla, la respuesta es `400 POLITICA_CONTRASENA` y `Motivos` lista una o varias de estas claves:

| Clave | Significado |
|---|---|
| `Vacia` | No se envió contraseña |
| `Corta` / `Larga` | Menos de 8 o más de 256 caracteres |
| `SinMinuscula` / `SinMayuscula` / `SinNumero` / `SinSimbolo` | Falta ese tipo de carácter |
| `EspaciosExtremos` | Empieza o termina con espacio |
| `CaracteresInvalidos` | Lleva caracteres de control |

## Quién puede qué

| Acción | Roles |
|---|---|
| Crear usuarios y editar sus datos | `Administrador`, `Comercial`, `CEO`, `Desarrollador` |
| Dar o cambiar roles y tiendas de usuarios **existentes** | `Administrador`, `Desarrollador` |
| Al **crear**, asignar roles que no son de administración | Todos los de la primera fila |
| Al crear, asignar `Administrador`, `Comercial`, `CEO` o `Desarrollador` | Solo `Administrador` y `Desarrollador` |
| Cambiar la propia contraseña | Cualquier usuario autenticado |
| Editar los propios datos de contacto | Cualquier usuario autenticado |

Las listas salen de las variables `ROLES_ADMINISTRACION` y `ROLES_GESTION_PRIVILEGIOS` de cada lambda; si faltan, nadie puede nada. Los nombres de rol se comparan exactos (sin ignorar mayúsculas).

---

## `POST /createUser` — crear un usuario

Requiere header `Token` de una persona con rol de administración.

**Body** (la misma forma que ya envía la vista de usuarios):

```json
{
  "Usuario": {
    "email": "nuevo@tienda.com",
    "numTelefono": "3001234567",
    "tipoDocumento": "CC",
    "documento": "123456789",
    "autorizaDatos": true,
    "roles": ["Tienda"],
    "tiendas": ["TIENDA1"]
  },
  "Contrasena": "Segura#2026x",
  "TipoUsuario": "Tienda"
}
```

- `numTelefono` y `documento` pueden ir como texto o número.
- El `username` que mande el front **se ignora**: se calcula como el prefijo del correo, en minúsculas.
- Al menos un rol. Ningún texto pasa de 100 caracteres y el correo no pasa de 254.

| Código HTTP | `Codigo` | Cuándo |
|---|---|---|
| 200 | `OK` | Usuario creado en Cognito (contraseña permanente) y en `Users` |
| 400 | `DATOS_INVALIDOS` | Faltan datos o el cuerpo no es válido |
| 400 | `POLITICA_CONTRASENA` | La contraseña no cumple (ver `Motivos`) |
| 401 | `NO_AUTORIZADO` | Sin `Token`, falso o vencido |
| 403 | `PROHIBIDO` | Sin registro en `Users` o sin rol de administración |
| 403 | `ROL_NO_PERMITIDO` | Pidió un rol que su rol no puede asignar (`Motivos` lista los rechazados) |
| 409 | `USUARIO_DUPLICADO` | Ya existe una cuenta con ese correo |
| 500 | `ERROR_INTERNO` | Fallo inesperado. Si el guardado en `Users` falla, la cuenta de Cognito se elimina para no dejar usuarios huérfanos |

No se envía correo a la persona creada: `createUser` no manda mensajes. Cognito no le pide cambiar la contraseña en el primer ingreso.

## `POST /cambio-pass` — cambiar la propia contraseña

Requiere header `Token`. Solo cambia **la contraseña de quien tiene la sesión**; el correo del cuerpo ya no existe.

```json
{ "ContrasenaActual": "La que tiene hoy", "Contrasena": "Nueva#2026x" }
```

| Código HTTP | `Codigo` | Cuándo |
|---|---|---|
| 200 | `OK` | Contraseña cambiada |
| 400 | `DATOS_INVALIDOS` | Falta alguna contraseña |
| 400 | `POLITICA_CONTRASENA` | La nueva no cumple |
| 400 | `NUEVA_IGUAL_ACTUAL` | La nueva es igual a la actual |
| 401 | `NO_AUTORIZADO` | Token ausente, falso o vencido |
| 403 | `CLAVE_ACTUAL_INVALIDA` | La contraseña actual no es correcta |
| 429 | `DEMASIADOS_INTENTOS` | Demasiados intentos fallidos; esperar unos minutos |
| 500 | `ERROR_INTERNO` | Fallo inesperado |

!!! danger "No registrar este cuerpo"
    El cuerpo lleva dos contraseñas. El front **no debe** mandarlo a la auditoría genérica (`insertarGenerico` copia el cuerpo a la colección de auditoría). La ruta `cambio-pass` ya está excluida en `RUTAS_SIN_AUDITORIA`.

## `/users` — usuarios (GET, POST, PUT)

Todos los métodos requieren header `Token` y que la persona tenga registro en `Users`.

**`GET /users?page=1&email=...`**: sin cambios en la respuesta (`Resultado` comprimido en gzip y `NumeroPaginas`). Cualquier usuario autenticado puede consultar. Restringirlo por rol queda en el backlog (M-199) porque el front lo usa al iniciar sesión y en listados.

**`PUT /users?_id=<id>`** con un cuerpo parcial de campos a cambiar:

| Campo | Quién lo puede cambiar |
|---|---|
| `_id`, `cognitoId`, `email`, `username`, `createdAt` | Nadie por esta ruta (`400 CAMPOS_PROTEGIDOS`) |
| `roles`, `tiendas`, `idtiendas`, `idMensajero` | Roles de gestión de privilegios, y nunca sobre sí mismos (`403 PRIVILEGIOS_NO_PERMITIDOS`) |
| Cualquier otro | Roles de administración sobre cualquiera; cualquier usuario sobre su propio registro |

Nombres de campo que empiezan con `$` o llevan punto se rechazan (`400 DATOS_INVALIDOS`). Si el `_id` no existe responde `404 NO_ENCONTRADO`.

**`POST /users`**: queda para quien tenga rol de administración y solo con roles que pueda asignar. Un documento nuevo no puede reutilizar el `cognitoId` de una cuenta ya registrada (`409 USUARIO_DUPLICADO`). Para crear personas usa `/createUser`, que además crea la cuenta de Cognito.

## `POST /registerContact` — formulario de contacto (público)

Es el formulario de la landing y no exige sesión. Solo acepta `POST` (los demás métodos responden `405`).

- Cuerpo: un arreglo con 1 a 3 contactos, o un contacto suelto.
- Campos aceptados: `userName`, `email`, `phoneNumber`, `source`, `comienzo`, `city`, `NombreComercio`, `ventasPorMes`, `businessType` (texto de hasta 200 caracteres) e `isColombian` (booleano). Cualquier otro campo se descarta.
- Obligatorios: `userName`, `email` válido y `phoneNumber`.
- `fechaRegistro` y `estado` (`No Contactado`) los pone el servidor; lo que mande el front se ignora.
- Responde `200 OK`, `400 DATOS_INVALIDOS` o `500 ERROR_INTERNO`.

## `POST /obtenerTokenAuth` — token de servicio

La usan **servicios** (Hoko Wallet, Guías, Liquidaciones, intereses de apalancamiento), no el front. Contrato de éxito sin cambios: el cuerpo trae usuario y clave (la clave en Base64) y el token vuelve en el header `Token`. Cambios de seguridad sin romper el contrato: un cuerpo inválido responde `400`, los errores ya no devuelven el mensaje interno de Cognito y se retiró `Access-Control-Allow-Credentials`. La sustitución por credenciales de máquina (OAuth2 `client_credentials`) está en el backlog (M-194).

## Registro por autoservicio con OTP (*planeado*, spec 008)

Rutas públicas que **aún no existen**; este contrato es el que implementará el backend y el que usará el front para sus vistas.

| Ruta | Para qué |
|---|---|
| `POST /autenticacion/registro/solicitudes` | La persona envía sus datos; se le manda un código de 6 dígitos al correo |
| `POST /autenticacion/registro/solicitudes/reenvio` | Pide otro código (invalida el anterior) |
| `POST /autenticacion/registro/confirmaciones` | Envía el código y define su contraseña; se crea la cuenta |
| `POST /autenticacion/registro/aprobaciones/{usuarioId}` | Con token y rol: aprueba al usuario y le asigna roles y tiendas |

Detalle de campos, respuestas y reglas en la spec `008_registro_autoservicio` y en la [guía del front](../frontend/autenticacion.md).
