# Autenticación y registro: guía para el equipo de front

Qué cambió en el API de cuentas y qué vistas debe construir el front. Contrato completo de cada ruta (campos, respuestas y errores): [API de autenticación](../api/autenticacion.md). El backend está listo en **PreProd**; el front construye toda la experiencia (UX/UI) sobre estos endpoints y se prueba el flujo completo antes de producción.

## Antes de empezar: a qué apuntar en PreProd

El backend nuevo vive en la cuenta de **PreProd** de AWS. Para probar el flujo completo, la configuración de PreProd del front debe usar estos valores (no son secretos):

| Qué | Valor |
|---|---|
| Base del API de estas rutas | `https://8u5isw5u30.execute-api.us-east-1.amazonaws.com/PreProd` |
| `cognitoUserPoolId` | `us-east-1_NtFz4uKGk` |
| `cognitoAppClientId` | `37n1vom76u3godto2losctoh2v` |

!!! warning "Hoy `environment.preprod.ts` apunta a otros recursos"
    En la revisión del 2026-11-03 apuntaba a un user pool, un app client y un API (`g3iz6qk3f0`) que **no existen en la cuenta de PreProd**: una cuenta creada con estas rutas no puede iniciar sesión con esa configuración. Además, el API de PreProd tiene las rutas de autenticación pero hay que confirmar con backend si tiene las demás que usa el front (inventario, guías...). Si no, usen una URL base propia para las rutas de cuentas y dejen las demás como están.

**Correos de prueba.** SES de PreProd solo entrega a correos verificados o a direcciones `@logigho.com` con buzón existente. Si necesitan probar con un correo externo, pídanle a Iker que lo habilite. Los correos pueden llegar a *Correo no deseado* en Hotmail/Outlook.

## Qué vistas hay que construir

| # | Vista | Endpoints | Quién la usa |
|---|---|---|---|
| 1 | Registro (formulario) | `POST /autenticacion/registro/solicitudes` | Público |
| 2 | Código y contraseña (desde el enlace del correo) | `POST .../confirmaciones`, `POST .../solicitudes/reenvio` | Público |
| 3 | Ingreso | Cognito directo (sin cambios) | Todos |
| 4 | Olvidé mi contraseña / Cambiar contraseña (mismo flujo) | `POST /autenticacion/recuperacion/solicitudes`, `POST /autenticacion/recuperacion/confirmaciones` | Público (desde el ingreso) y usuario con sesión (desde el perfil) |
| 5 | Asignar acceso a un cliente (en `/app/comercial/clientes`) | `GET /autenticacion/roles-asignables`, `POST /autenticacion/usuarios/{usuarioId}/acceso` | Comercial, CEO, Administrador, Desarrollador |
| 6 | Crear y editar usuarios (ya existe) | `/createUser`, `/users` | Administrador, Desarrollador |

## Reglas que valen para todo el front

1. **La identidad va en el header `Token`** con el `id_token` de la sesión (`sessionStorage.getItem('id_token')`). `headersecurity` ya no autoriza nada en las rutas de cuentas.
2. **Nunca mandes a auditoría un cuerpo con contraseñas.** `ConsumoGenericoService.insertarGenerico` copia el cuerpo a la colección de auditoría salvo para las rutas de `RUTAS_SIN_AUDITORIA`. `cambio-pass` ya está en esa lista; agrega también `registro/confirmaciones` y `recuperacion/confirmaciones`, y cualquier otra ruta con contraseñas o códigos.
3. **Decide la UX con `Codigo`, no con el texto ni con el estado HTTP solo.** El cuerpo de error es `{ Error, Codigo, Mensaje, Motivos }`. `HttpErrorResponse.message` de Angular es genérico; el detalle está en `error.error.Mensaje`.
4. **No pidas el rol al servidor para decidir permisos de seguridad.** Ocultar un botón es UX; el servidor es quien manda.

## Cambios ya hechos (rama `feature/autenticacion-front`)

| Archivo | Cambio |
|---|---|
| `register-service.service.ts` | `createUser` manda `Token`. Los tres sitios que lo llaman no cambian |
| `cambio-contrasena.component.*` | Campo **Contraseña actual** y cuerpo `{ ContrasenaActual, Contrasena }`. **Hay que migrarlo**: el flujo nuevo no pide contraseña actual ni usa `/cambio-pass` (ver la vista 4). Desde el PR 577 el modal actual ya no funciona en PreProd |
| `consumo-generico.service.ts` | `cambio-pass` agregado a `RUTAS_SIN_AUDITORIA` |
| Pruebas | `register-service`, `consumo-generico` y `cambio-contrasena` cubren Token, cuerpo y no-auditoría |

!!! warning "Orden de despliegue"
    El front con `Token` en `createUser` y el backend nuevo de `createUser` **salen juntos**. Con el backend nuevo y el front viejo, crear usuarios responde `401`. Con el front nuevo y el backend viejo no pasa nada (el header sobra). Por eso en PreProd se prueba el par completo antes de producción.

## Tabla de errores y qué mostrar

| `Codigo` | Dónde | Mensaje sugerido |
|---|---|---|
| `NO_AUTORIZADO` (401) | Cualquier ruta de cuentas | La sesión venció: volver al login |
| `PROHIBIDO` (403) | `createUser`, `users` | "No tienes permiso para esta acción" |
| `ROL_NO_PERMITIDO` (403) | `createUser`, `POST users` | "No puedes asignar estos roles" y mostrar `Motivos` |
| `PRIVILEGIOS_NO_PERMITIDOS` (403) | `PUT users` | "Solo un administrador cambia roles y tiendas" |
| `POLITICA_CONTRASENA` (400) | `createUser`, confirmación del registro y de la recuperación | Marcar cada requisito de `Motivos` que falla (tabla en el contrato) |
| `DEMASIADOS_INTENTOS` (429) | confirmación del registro y de la recuperación | "Demasiados intentos": pedir un código nuevo o empezar de nuevo |
| `DEMASIADAS_SOLICITUDES` (429) | pedir/reenviar código | "Espera un momento antes de pedir otro código" (60 s entre envíos, 3 por hora) |
| `CODIGO_INVALIDO` (400) | confirmación del registro y de la recuperación | "El código no es correcto" (máximo 5 intentos) |
| `SOLICITUD_VENCIDA` / `SOLICITUD_NO_DISPONIBLE` (410) | registro y recuperación | "La solicitud venció" y volver a empezar. También sale si se mandó mal el `solicitudId` (por ejemplo la dirección completa en vez del parámetro) |
| `SOLICITUD_YA_USADA` (409) | registro y recuperación | "Este código ya se usó" |
| `CORREO_YA_REGISTRADO` (409) | registro | "Ya tienes una cuenta: ingresa" |
| `AUTO_ASIGNACION_NO_PERMITIDA` / `USUARIO_PROTEGIDO` (403) | asignar acceso | "No puedes cambiar el acceso de este usuario" |
| `NO_ENCONTRADO` (404) | asignar acceso | "Ese usuario ya no existe" |
| `USUARIO_DUPLICADO` (409) | `createUser` | "Ya existe un usuario con ese correo" |
| `CAMPOS_PROTEGIDOS` (400) | `PUT users` | Error de programación: el front no debe enviar `email`, `username`, `cognitoId`, `_id` ni `createdAt` |
| `ERROR_INTERNO` (500) | Todas | "No pudimos completar la acción; intenta de nuevo" |

## Pendiente del front (backlog del dominio)

- **Edición de roles y tiendas desde la vista de usuarios** (M-196): el `PUT` a `users` ya aplica la política; la vista debe ocultar el selector de roles de administración a quien no los puede asignar y manejar `403`.
- **`accessToken` real** (no `id_token`) en la sesión cuando se migre el authorizer del gateway (spec 009).
- Las vistas 1, 2, 4 y 5 de la tabla del inicio (abajo el detalle).

## Vista 1 y 2. Registro por autoservicio

### Pantallas

1. **Formulario.** Los campos de hoy del contacto más la autorización de datos. El front puede seguir guardando el contacto con `/registerContact` como hoy y luego llamar a `POST /autenticacion/registro/solicitudes` con los mismos datos; al confirmar, la fila se actualiza a "Registrado en plataforma" **sin duplicarse**. La solicitud responde **siempre 202** con el mismo mensaje exista o no el correo: la pantalla no puede decir "ese correo ya existe". Pasa a la pantalla 2 ("Revisa tu correo").
2. **Código y contraseña.** Se abre desde el enlace del correo `{URL_REGISTRO}?solicitud=<solicitudId>`. La pantalla vive en `/app/registro` (acordado con el front el 2026-11-04); si la ruta cambia, avisen para cambiar la variable. El `solicitudId` viaja **solo en la URL**: toma **solo el valor del parámetro** `solicitud` (`route.snapshot.queryParamMap.get('solicitud')`), nunca la dirección completa (con la dirección completa el servidor responde `SOLICITUD_VENCIDA`); guárdalo en memoria, no en `localStorage`, y bórralo de la barra con `history.replaceState`. Campos: código de 6 dígitos y contraseña (con confirmación y la política del contrato). `POST .../confirmaciones`.
3. **Reenvío.** Botón "Reenviar código" deshabilitado 60 segundos; `POST .../solicitudes/reenvio` con el `solicitudId`.
4. **Resultado.** `201` con `estado: "ACTIVO"`: ya puede iniciar sesión con su correo y su contraseña. Entra con el rol **`Red social`** (solo la red social) hasta que Comercial le asigne el acceso real. El menú que ve debe salir de sus roles como hoy.

### Respuestas a manejar

| Estado | `Codigo` | Qué mostrar |
|---|---|---|
| 202 | `OK` | "Si el correo puede registrarse, recibirás un código" |
| 400 | `DATOS_INVALIDOS` | Resaltar campos; `autorizaDatos` debe ser `true` |
| 400 | `POLITICA_CONTRASENA` | Requisitos que fallan; **no gasta intento** |
| 400 | `CODIGO_INVALIDO` | "Código incorrecto" (máximo 5 intentos) |
| 410 | `SOLICITUD_VENCIDA` | "El código venció" y ofrecer volver a solicitar |
| 429 | `DEMASIADAS_SOLICITUDES` / `DEMASIADOS_INTENTOS` | "Espera unos minutos" / "empieza de nuevo" |

### Reglas de UX y seguridad

- El código dura **15 minutos**; muestra una cuenta regresiva y no la uses como regla (la regla la aplica el servidor).
- Entrada de código: `inputmode="numeric"`, `autocomplete="one-time-code"`, 6 casillas o un campo con máscara.
- Contraseña: `autocomplete="new-password"`; no la guardes ni la registres en consola.
- No incluyas el correo ni el código en la URL, en analítica ni en mensajes de error.
- Agrega las rutas con contraseñas o códigos a `RUTAS_SIN_AUDITORIA` (`cambio-pass`, `cambio-pass/codigo` y las de registro).

## Vista 4. Olvidé mi contraseña y Cambiar contraseña (un solo flujo)

Decisión 2026-11-03: **no se pide la contraseña actual**. El factor de seguridad es un código de 6 dígitos que llega al correo de la cuenta, y es el mismo servicio para quien olvidó su contraseña y para quien la cambia desde el perfil. Las rutas `/cambio-pass` y `/cambio-pass/codigo` quedan **reemplazadas**: **migren el modal de cambiar contraseña a este flujo**. Las rutas viejas se eliminarán cuando termine la migración.

Pantallas:

1. **Pedir el código.** Campo de correo (en el perfil ya se conoce: ofrece "Enviar código a tu correo" con el correo enmascarado). `POST /autenticacion/recuperacion/solicitudes` con `{ "email": "..." }`. Responde **siempre 202** con el mismo mensaje: la pantalla dice "Si existe una cuenta con ese correo, te enviamos un código" y nunca confirma si el correo existe.
2. **Código y contraseña nueva.** Se abre desde el enlace del correo `{URL_RECUPERACION}?solicitud=<solicitudId>` (`https://www.logigho.com/app/recuperar`, acordado con el front el 2026-11-04; si la ruta cambia, avisen para cambiar la variable). Toman **solo el valor** del parámetro `solicitud`, lo guardan en memoria y lo quitan de la barra. Campos: código de 6 dígitos y contraseña nueva con confirmación (política del contrato). `POST /autenticacion/recuperacion/confirmaciones` con `{ "solicitudId", "codigo", "contrasena" }`.
3. **Resultado.** `200`: la contraseña cambió y **se cerraron todas las sesiones** del usuario. Cierra la sesión local y manda a la pantalla de ingreso. Le llega además un correo de aviso del cambio.
4. **Reenviar código.** Vuelve a llamar al paso 1 con el correo (60 s de espera, máximo 3 por hora). Los códigos anteriores siguen valiendo hasta que venzan; se debe usar el correo más reciente.

Respuestas a manejar: `202 OK` (siempre en el paso 1); en el paso 2, `400 POLITICA_CONTRASENA` (marcar los `Motivos`, no gasta intento), `400 CODIGO_INVALIDO` (máximo 5 intentos), `409 SOLICITUD_YA_USADA`, `410 SOLICITUD_VENCIDA` (pedir otro código) y `429 DEMASIADOS_INTENTOS` (pedir otro código). Los textos sugeridos están en la tabla de errores de arriba.

Reglas de UX y seguridad: las mismas del registro (entrada numérica con `autocomplete="one-time-code"`, contraseña con `autocomplete="new-password"`, sin correo ni código en URL, analítica o errores, y rutas en `RUTAS_SIN_AUDITORIA`).

Nota: las cuentas creadas con una red social (Google, etc.) no tienen contraseña propia; el cambio de contraseña no aplica y hoy responde `500`. Si el front ofrece ingreso con redes, oculten "Cambiar contraseña" para esas cuentas hasta que se defina el comportamiento.

## Vista 5. Asignar acceso desde `/app/comercial/clientes`

Hoy esa vista lista `RegisterContact`. Al registrarse una persona, su fila queda con `estado: "Registrado en plataforma"` y un `userId`. Comercial no tiene acceso a la vista de usuarios, así que el front agrega aquí la acción **"Asignar acceso"** para filas con `userId`:

1. Al abrir el diálogo: `GET /autenticacion/roles-asignables` (con `Token`). Llena el selector con `Roles`. Lo que no venga en la lista no se ofrece (si Comercial ve solo 4 roles, el servidor también los limita a esos).
2. Al guardar: `POST /autenticacion/usuarios/{userId}/acceso` con `{ "roles": [...], "tiendas": [...] }`. `userId` es el de la fila.
3. Con `200`, refrescar la fila. El usuario deja de tener `Red social` y pasa a tener los roles asignados.
4. Manejar `403 ROL_NO_PERMITIDO` (mostrar `Motivos`), `403 USUARIO_PROTEGIDO`, `404` y `401`.

El front **no decide** quién puede qué: oculta lo que no corresponde por UX, pero el servidor es quien manda.

## Estado de la migración (2026-11-04)

- La rama `feature/inicio-sesion-la-red` ya usa **solo** las dos rutas de recuperación para recuperar y cambiar la contraseña; ya no llama a `/cambio-pass`, `/cambio-pass/codigo` ni a Cognito directo para eso. Cuando sus pruebas pasen, backend elimina las rutas viejas (M-232).
- El login ya no tiene botones de redes sociales. **Pendiente de acordar:** el mensaje para las cuentas creadas antes con una red social, que hoy reciben `500` al confirmar la recuperación (M-229).

## Un usuario con solo `Red social` en el front

Quien se registra por autoservicio queda con el rol `Red social` hasta que Comercial le asigne acceso. Con ese rol **no puede administrar nada**: `GET /autenticacion/roles-asignables` responde `403 PROHIBIDO` y `GET /users` le devuelve **únicamente su propio registro** (sin importar el filtro o la página que mande). Si una pantalla de listados llama a `/users` para ese rol, solo verá su fila.
