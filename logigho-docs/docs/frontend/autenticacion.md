# Autenticación y registro: guía para el equipo de front

Qué cambió en el API de cuentas y qué vistas debe construir el front. Contrato completo de cada ruta (campos, respuestas y errores): [API de autenticación](../api/autenticacion.md). El backend está listo en **PreProd**; el front construye toda la experiencia (UX/UI) sobre estos endpoints y se prueba el flujo completo antes de producción.

## Qué vistas hay que construir

| # | Vista | Endpoints | Quién la usa |
|---|---|---|---|
| 1 | Registro (formulario) | `POST /autenticacion/registro/solicitudes` | Público |
| 2 | Código y contraseña (desde el enlace del correo) | `POST .../confirmaciones`, `POST .../solicitudes/reenvio` | Público |
| 3 | Ingreso | Cognito directo (sin cambios) | Todos |
| 4 | Cambio de contraseña con código | `POST /cambio-pass/codigo`, `POST /cambio-pass` | Usuario con sesión |
| 5 | Asignar acceso a un cliente (en `/app/comercial/clientes`) | `GET /autenticacion/roles-asignables`, `POST /autenticacion/usuarios/{usuarioId}/acceso` | Comercial, CEO, Administrador, Desarrollador |
| 6 | Crear y editar usuarios (ya existe) | `/createUser`, `/users` | Administrador, Desarrollador |

## Reglas que valen para todo el front

1. **La identidad va en el header `Token`** con el `id_token` de la sesión (`sessionStorage.getItem('id_token')`). `headersecurity` ya no autoriza nada en las rutas de cuentas.
2. **Nunca mandes a auditoría un cuerpo con contraseñas.** `ConsumoGenericoService.insertarGenerico` copia el cuerpo a la colección de auditoría salvo para las rutas de `RUTAS_SIN_AUDITORIA`. `cambio-pass` ya está en esa lista; si agregas otra ruta con contraseñas o códigos, agrégala también.
3. **Decide la UX con `Codigo`, no con el texto ni con el estado HTTP solo.** El cuerpo de error es `{ Error, Codigo, Mensaje, Motivos }`. `HttpErrorResponse.message` de Angular es genérico; el detalle está en `error.error.Mensaje`.
4. **No pidas el rol al servidor para decidir permisos de seguridad.** Ocultar un botón es UX; el servidor es quien manda.

## Cambios ya hechos (rama `feature/autenticacion-front`)

| Archivo | Cambio |
|---|---|
| `register-service.service.ts` | `createUser` manda `Token`. Los tres sitios que lo llaman no cambian |
| `cambio-contrasena.component.*` | Campo **Contraseña actual**; el cuerpo es `{ ContrasenaActual, Contrasena }` (sin correo); el modal ya no se cierra antes de saber el resultado; el error del servidor se muestra. **Falta** el paso del código: ver la vista 4 de abajo |
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
| `POLITICA_CONTRASENA` (400) | `createUser`, `cambio-pass`, confirmación OTP | Marcar cada requisito de `Motivos` que falla (tabla en el contrato) |
| `CLAVE_ACTUAL_INVALIDA` (403) | `cambio-pass` | "La contraseña actual no es correcta" |
| `NUEVA_IGUAL_ACTUAL` (400) | `cambio-pass` | "Elige una contraseña distinta" |
| `DEMASIADOS_INTENTOS` (429) | `cambio-pass`, confirmación del registro | "Demasiados intentos": pedir un código nuevo o empezar de nuevo |
| `DEMASIADAS_SOLICITUDES` (429) | pedir/reenviar código | "Espera un momento antes de pedir otro código" (60 s entre envíos, 3 por hora) |
| `CODIGO_INVALIDO` (400) | `cambio-pass`, confirmación del registro | "El código no es correcto" (máximo 5 intentos) |
| `CODIGO_VENCIDO` (410) | `cambio-pass` | "El código venció" y ofrecer pedir otro |
| `SOLICITUD_VENCIDA` / `SOLICITUD_NO_DISPONIBLE` (410) | registro | "La solicitud venció" y volver a empezar |
| `SOLICITUD_YA_USADA` / `CORREO_YA_REGISTRADO` (409) | registro | "Ya tienes una cuenta: ingresa" |
| `CUENTA_SIN_CORREO` (409) | `cambio-pass/codigo` | "Tu cuenta no tiene correo registrado; comunícate con soporte" |
| `CORREO_NO_ENVIADO` (503) | `cambio-pass/codigo` | "No pudimos enviar el código; intenta de nuevo" |
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
2. **Código y contraseña.** Se abre desde el enlace del correo `{URL_REGISTRO}?solicitud=<solicitudId>`. Hay que definir la ruta final de esta pantalla y avisar para cambiar la variable (hoy `https://www.logigho.com/registro`). El `solicitudId` viaja **solo en la URL**: guárdalo en memoria, no en `localStorage`, y bórralo de la barra con `history.replaceState`. Campos: código de 6 dígitos y contraseña (con confirmación y la política del contrato). `POST .../confirmaciones`.
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

## Vista 4. Cambio de contraseña con código

Con sesión iniciada, en dos pasos (decisión M-214: contraseña actual **y** código al correo de la cuenta):

1. El usuario pulsa "Cambiar contraseña". El front llama `POST /cambio-pass/codigo` (con `Token`, sin cuerpo) y guarda en memoria el `solicitudId` de la respuesta. Se le dice "Te enviamos un código a tu correo" (puedes enmascarar el correo en la pantalla con el que ya conoce el front; el servidor no lo devuelve).
2. El usuario escribe **contraseña actual, contraseña nueva y el código**. El front llama `POST /cambio-pass` con `{ ContrasenaActual, Contrasena, Codigo, SolicitudId }`.
3. Con `200` se **cierra la sesión local** y se pide ingresar con la contraseña nueva (el servidor revoca los refresh tokens).
4. "Reenviar código": vuelve a llamar el paso 1; hay 60 s de espera y máximo 3 por hora.

Un `403 CLAVE_ACTUAL_INVALIDA` no consume el código: puede corregir la contraseña actual y reintentar con el mismo código.

## Vista 5. Asignar acceso desde `/app/comercial/clientes`

Hoy esa vista lista `RegisterContact`. Al registrarse una persona, su fila queda con `estado: "Registrado en plataforma"` y un `userId`. Comercial no tiene acceso a la vista de usuarios, así que el front agrega aquí la acción **"Asignar acceso"** para filas con `userId`:

1. Al abrir el diálogo: `GET /autenticacion/roles-asignables` (con `Token`). Llena el selector con `Roles`. Lo que no venga en la lista no se ofrece (si Comercial ve solo 4 roles, el servidor también los limita a esos).
2. Al guardar: `POST /autenticacion/usuarios/{userId}/acceso` con `{ "roles": [...], "tiendas": [...] }`. `userId` es el de la fila.
3. Con `200`, refrescar la fila. El usuario deja de tener `Red social` y pasa a tener los roles asignados.
4. Manejar `403 ROL_NO_PERMITIDO` (mostrar `Motivos`), `403 USUARIO_PROTEGIDO`, `404` y `401`.

El front **no decide** quién puede qué: oculta lo que no corresponde por UX, pero el servidor es quien manda.
