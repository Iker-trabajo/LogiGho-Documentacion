# Autenticación y registro: guía para el equipo de front

Qué cambió en el API de cuentas, qué ya hizo el front en la rama `feature/autenticacion-front` y qué falta. Contrato completo de cada ruta: [API de autenticación](../api/autenticacion.md).

## Reglas que valen para todo el front

1. **La identidad va en el header `Token`** con el `id_token` de la sesión (`sessionStorage.getItem('id_token')`). `headersecurity` ya no autoriza nada en las rutas de cuentas.
2. **Nunca mandes a auditoría un cuerpo con contraseñas.** `ConsumoGenericoService.insertarGenerico` copia el cuerpo a la colección de auditoría salvo para las rutas de `RUTAS_SIN_AUDITORIA`. `cambio-pass` ya está en esa lista; si agregas otra ruta con contraseñas o códigos, agrégala también.
3. **Decide la UX con `Codigo`, no con el texto ni con el estado HTTP solo.** El cuerpo de error es `{ Error, Codigo, Mensaje, Motivos }`. `HttpErrorResponse.message` de Angular es genérico; el detalle está en `error.error.Mensaje`.
4. **No pidas el rol al servidor para decidir permisos de seguridad.** Ocultar un botón es UX; el servidor es quien manda.

## Cambios ya hechos (rama `feature/autenticacion-front`)

| Archivo | Cambio |
|---|---|
| `register-service.service.ts` | `createUser` manda `Token`. Los tres sitios que lo llaman no cambian |
| `cambio-contrasena.component.*` | Campo **Contraseña actual**; el cuerpo es `{ ContrasenaActual, Contrasena }` (sin correo); el modal ya no se cierra antes de saber el resultado; el error del servidor se muestra |
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
| `DEMASIADOS_INTENTOS` (429) | `cambio-pass` | "Espera unos minutos y vuelve a intentar" |
| `USUARIO_DUPLICADO` (409) | `createUser` | "Ya existe un usuario con ese correo" |
| `CAMPOS_PROTEGIDOS` (400) | `PUT users` | Error de programación: el front no debe enviar `email`, `username`, `cognitoId`, `_id` ni `createdAt` |
| `ERROR_INTERNO` (500) | Todas | "No pudimos completar la acción; intenta de nuevo" |

## Pendiente del front (backlog del dominio)

- **Edición de roles y tiendas desde la vista de usuarios** (M-196): hoy el `PUT` a `users` ya aplica la política; la vista debe ocultar el selector de roles de administración a quien no los puede asignar y manejar `403`.
- **`accessToken` real** (no `id_token`) en la sesión cuando se migre el authorizer del gateway (spec 009).
- **Vistas del registro por OTP** (abajo).

## Registro por autoservicio con OTP (*planeado*, spec 008)

Backend **aún no desplegado**. Este es el contrato para construir las vistas; mientras no exista, el flujo se prueba con Postman (ver [guía de pruebas](../guias/pruebas-autenticacion-e2e.md)).

### Pantallas

1. **Solicitud.** Formulario con los campos de hoy del contacto más la autorización de datos. `POST /autenticacion/registro/solicitudes` responde **siempre 202** con el mismo mensaje exista o no el correo: la pantalla no puede decir "ese correo ya existe". Pasa a la pantalla 2.
2. **Código y contraseña.** Se abre desde el enlace del correo `{URL_REGISTRO}?solicitud=<solicitudId>`. El `solicitudId` viaja **solo en la URL**: guárdalo en memoria, no en `localStorage`, y bórralo de la barra con `history.replaceState`. Campos: código de 6 dígitos y contraseña (con confirmación y la política de arriba). `POST .../confirmaciones`.
3. **Reenvío.** Botón "Reenviar código" deshabilitado 60 segundos; `POST .../solicitudes/reenvio` con el `solicitudId`.
4. **Resultado.** `201` con `estado`: `ACTIVO` (ya puede iniciar sesión) o `PENDIENTE_APROBACION` (un administrador debe aprobarlo; mostrar "te avisaremos").

### Respuestas a manejar

| Estado | `Codigo` | Qué mostrar |
|---|---|---|
| 202 | — | "Si el correo puede registrarse, recibirás un código" |
| 400 | `DATOS_INVALIDOS` | Resaltar campos; `autorizaDatos` debe ser `true` |
| 400 | `POLITICA_CONTRASENA` | Requisitos que fallan; **no gasta intento** |
| 400 | `CODIGO_INVALIDO` | "Código incorrecto" (máximo 5 intentos) |
| 410 | `SOLICITUD_VENCIDA` | "El código venció" y ofrecer volver a solicitar |
| 429 | `DEMASIADAS_SOLICITUDES` / `DEMASIADOS_INTENTOS` | "Espera unos minutos" |

### Reglas de UX y seguridad

- El código dura **15 minutos**; muestra una cuenta regresiva y no la uses como regla (la regla la aplica el servidor).
- Entrada de código: `inputmode="numeric"`, `autocomplete="one-time-code"`, 6 casillas o un campo con máscara.
- Contraseña: `autocomplete="new-password"`; no la guardes ni la registres en consola.
- No incluyas el correo ni el código en la URL, en analítica ni en mensajes de error.

### Vistas mínimas para pruebas

Mientras el equipo diseña las vistas finales, se puede probar el flujo completo con **Postman** (colección en la guía de pruebas) y, si se necesita ver el recorrido en pantalla, con una página HTML estática de dos formularios que llame a las mismas rutas. No hay que construir nada en Angular para validar el backend.
