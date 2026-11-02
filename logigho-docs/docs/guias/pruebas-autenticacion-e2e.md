# Pruebas punta a punta del dominio autenticación

Guía para probar en **PreProd** todo lo que cambió, antes de tocar producción. Se hace con Postman (colección incluida) y no necesita vistas de Angular.

Descargas: [colección](autenticacion-e2e.postman_collection.json) · [environment](autenticacion-e2e.postman_environment.json). Ambos traen variables vacías: **no guardes contraseñas en el archivo ni en git**; llénalas solo en tu Postman local (las marcadas `secret` no se sincronizan).

## 0. Qué debe existir antes de empezar

| Requisito | Quién | Cómo se comprueba |
|---|---|---|
| Lambdas con el código nuevo publicadas en PreProd (runtime `dotnet10` salvo `registerContact` y `obtenerTokenAuth`, que siguen net8 AOT) | Iker | `aws lambda get-function-configuration` muestra el runtime y la fecha |
| Variables de entorno aplicadas desde la Library (ver [backend](../backend/autenticacion.md)) | Iker | Sin `MONGO_URI` el createUser responde `500` |
| Rutas `/createUser`, `/cambio-pass`, `/users`, `/registerContact`, `/obtenerTokenAuth` en el REST de PreProd, **sin authorizer** (la lambda valida el `Token`) y stage desplegado | Iker / Terraform | Un `OPTIONS` a cada ruta responde 200 |
| User pool de PreProd con los mismos app clients de las variables `COGNITO_CLIENT_IDS` | Iker | `COGNITO_USER_POOL_ID` coincide con el pool |
| Tres usuarios de prueba **creados en PreProd**: uno `Administrador`, uno `Comercial` y uno `Tienda`, con registro en `Users` y su `cognitoId` | Iker | La colección los usa para iniciar sesión |
| Front de PreProd con la rama `feature/autenticacion-front` (para la prueba manual del paso 4) | Iker | El modal de cambio de contraseña muestra "Contraseña actual" |

!!! danger "No uses producción para esta prueba"
    `createUser` crea cuentas reales y `registerContact` guarda contactos. En PreProd los datos son de prueba. Los correos de prueba deben ser dominios que controlas (`@example.com` no recibe correo, sirve para estas pruebas porque `createUser` no envía mensajes).

## 1. Preparar Postman

1. Importa la colección y el environment. Selecciona el environment `autenticacion-preprod`.
2. Llena `baseUrl` (por ejemplo `https://<rest>.execute-api.us-east-1.amazonaws.com/<stage>`, sin barra final), las credenciales de los tres usuarios y `headersecurity` (es el mismo valor del `environment.preprod.ts` del front).
3. Elige `claveNueva` y `claveNueva2` que cumplan la política (8 o más caracteres con mayúscula, minúscula, número y símbolo) y sean distintas.
4. `tiendaPrueba`: un código de tienda que exista en PreProd.
5. Deja `idUsuarioNuevo` e `idOtroUsuario` para el paso 3: los pegas cuando el usuario nuevo exista.

## 2. Orden de ejecución

La colección se corre **en orden** con el Runner (`Run collection`). Cada petición trae sus pruebas y debe quedar en verde.

| Carpeta | Qué demuestra |
|---|---|
| 0. Tokens | Los tres usuarios de prueba obtienen token con `/obtenerTokenAuth` (el mismo camino que usan los servicios) |
| 1. createUser | Sin `Token` o con token falso responde 401 aunque el `headersecurity` sea correcto; un rol sin permiso responde 403; Comercial no puede asignar `Administrador`; contraseña débil 400 con motivos; creación correcta 200; correo repetido 409; el usuario nuevo **inicia sesión** |
| 2. cambio-pass | Sin sesión 401; actual incorrecta 403; nueva débil o igual 400; cambio correcto 200 y el login con la nueva funciona |
| 3. users | Sin token 401; un Tienda no crea ni edita a otros; Comercial no cambia roles; nadie cambia `email`/`cognitoId`; un operador `$set` se rechaza; un usuario edita su teléfono pero no el de otro |
| 4. registerContact | POST válido 200; incompleto o con correo inválido 400; `GET`, `PUT` y `DELETE` ya no existen (405) |
| 5. obtenerTokenAuth | Cuerpo vacío o clave mal codificada 400; credenciales incorrectas 500 sin revelar mensajes de Cognito |

!!! note "Entre la carpeta 1 y la 3"
    Después de crear el usuario nuevo, busca su `_id` con el script de lectura de la base (`tools/db-lectura/consulta_lectura.py`, `--entorno preprod`, con `--proyeccion`) y pégalo en `idUsuarioNuevo`. En `idOtroUsuario` pon el `_id` de cualquier otro usuario de prueba.

## 3. Qué revisar además de los verdes

1. **Cognito** (consola de PreProd): el usuario nuevo existe, `email_verified = true`, estado **CONFIRMED** (no `FORCE_CHANGE_PASSWORD`).
2. **Base `Users`**: un solo documento con el correo en minúsculas, `username` igual al prefijo del correo, `cognitoId` con el `sub` y `createdAt` del servidor. Sin campo de contraseña.
3. **Base `RegisterContact`**: el contacto de prueba, con `estado = "No Contactado"` y `fechaRegistro` del servidor aunque hayas intentado mandar otros valores.
4. **CloudWatch Logs** de cada lambda: líneas `[INFO]/[WARN]/[ERROR]` con resultado. Busca el correo, la contraseña y el token de prueba: **no deben aparecer** en ningún log.
5. **Auditoría**: tras cambiar la contraseña desde el front, la colección de auditoría no debe tener ningún registro de `cambio-pass`.

## 4. Prueba manual con el front de PreProd

| Acción | Resultado esperado |
|---|---|
| Entrar como `Comercial` > Usuarios > crear un usuario `Tienda` | Éxito; el usuario aparece en la lista; puede iniciar sesión |
| Mismo flujo escogiendo rol `Administrador` | Error claro: no puedes asignar ese rol |
| Entrar como el usuario nuevo > menú > Cambiar contraseña, con la actual equivocada | "La contraseña actual no es correcta"; el modal sigue abierto |
| Mismo con la actual correcta y una nueva válida | Éxito; cierra sesión, entra con la nueva |
| Landing > formulario de contacto | Éxito; el contacto aparece en la vista de clientes de Comercial |
| Con la sesión vencida (borra `id_token` de `sessionStorage`) intentar crear un usuario | El front pide iniciar sesión de nuevo |

## 5. Pruebas de reversión

1. Anota la versión actual de cada lambda **antes** de publicar.
2. Tras la prueba positiva, apunta el alias de una lambda a su versión anterior y repite una petición: debe volver al comportamiento viejo.
3. Vuelve a la versión nueva. Esto confirma que el rollback toma minutos.

## 6. Criterios para pasar a producción

- [ ] Toda la colección en verde en PreProd.
- [ ] Revisión de Cognito, `Users`, `RegisterContact`, logs y auditoría (sección 3) sin hallazgos.
- [ ] Prueba manual del front (sección 4) sin hallazgos.
- [ ] Front y backend de `createUser` se publican **en el mismo release**.
- [ ] Reversión probada (sección 5).
- [ ] Variables de producción cargadas en la Library y runtime `dotnet10` en las tres lambdas nuevas.
- [ ] Iker da la instrucción explícita de producción. Después: observar una semana (`[ERROR]`, 401 y 403 inesperados) antes de cerrar la siguiente ruta.

## 7. Registro con código, cambio de contraseña con código y asignación de acceso

Pruebas en **PreProd** (`https://8u5isw5u30.execute-api.us-east-1.amazonaws.com/PreProd`). La columna *Resultado* es lo observado el **2026-11-02**; *Pendiente* son las pruebas que necesitan leer el correo o un token de Comercial/Administrador y se hacen con Bruno o Postman con los pasos de la columna *Cómo*.

!!! note "SES en sandbox"
    Solo llegan correos a direcciones verificadas: `ikeracevedowrk+registro1@gmail.com`, `+registro2` y `+prueba1tienda`. Cada correo admite 3 solicitudes por hora; para repetir una prueba se usa otro alias o se espera la hora.

### 7.1 Ejecutadas

| # | Prueba | Cómo | Esperado | Resultado | Estado |
|---|---|---|---|---|---|
| 1 | Preflight CORS del registro | `OPTIONS /autenticacion/registro/solicitudes` con `Origin` | 200 y cabeceras `Access-Control-*` | 200 con `Allow-Origin: *` | OK |
| 2 | Solicitud con cuerpo vacío | `POST .../solicitudes` con `{}` | 400 `DATOS_INVALIDOS` | 400 `DATOS_INVALIDOS` | OK |
| 3 | Solicitud que no es JSON | `POST .../solicitudes` con `hola` | 400 | 400 | OK |
| 4 | Sin autorización de datos | `autorizaDatos: false` | 400 | 400 | OK |
| 5 | Confirmar solicitud inexistente | `POST .../confirmaciones` con `solicitudId` falso | 410 `SOLICITUD_VENCIDA` | 410 `SOLICITUD_VENCIDA` (sin dar pistas) | OK |
| 6 | Reenviar solicitud inexistente | `POST .../reenvio` | 410 `SOLICITUD_NO_DISPONIBLE` | 410 | OK |
| 7 | Solicitud válida (`registro1`) | formulario completo | 202; en base una fila `PENDIENTE` con firma, sin código, con `borrarEn` a +24 h | 202; fila con `otpHmac`, `otpClaveId`, `intentos 0`, TTL +24 h | OK |
| 8 | El correo se envía | misma solicitud, log de la lambda | `registro.otp.enviado` | **Falló dos veces** (`registro.ses.fallo`): SES en sandbox exige permiso sobre la identidad del destinatario. Se corrigió la política (Terraform) y a la tercera: `registro.otp.enviado` | Corregido |
| 9 | Límite por correo | 4.ª solicitud de `registro1` en la hora | 202 igual, sin fila nueva, evento `registro.solicitud.limitada` | 202; siguen 3 filas; evento registrado | OK |
| 10 | Correo que ya tiene cuenta | solicitud con `prueba1tienda` | 202 idéntico, **no** se crea solicitud (se envía el aviso de cuenta existente) | 202; 0 filas creadas | OK |
| 11 | Rutas protegidas sin token y con token falso | `GET roles-asignables`, `GET users`, `POST createUser`, `POST cambio-pass` | 401 en ambos casos | 401 / 401 en las cuatro | OK |
| 12 | Asignar acceso sin token | `POST /autenticacion/usuarios/{id}/acceso` con `{}` | 401 | **400** (validaba el cuerpo antes que la identidad). Corregido en el PR 577 | Corregido |
| 13 | Prueba de humo del CD | `curl -X GET -d '{}'` a `/users` | 401 | **403**: CloudFront rechaza un GET con cuerpo antes de llegar a la lambda. GET sin cuerpo = 401. Corregido en el PR 576 | Corregido |
| 14 | Rol `Red social` existe | lectura de `Roles` en PreProd y producción | existe, activo | existe en ambos (`Usuario con unico permiso a la red social`) | OK |
| 15 | Índices de `RegistroSolicitudes` | creados en PreProd | `solicitudHash` único, `email`+`creadoEn`, TTL `borrarEn` | 3 índices creados | OK |

### 7.2 Pendientes (necesitan el correo o un token de Comercial/Administrador)

Se hacen en este orden, con `registro2` y `prueba1tienda` para no agotar el límite de `registro1`.

| # | Prueba | Cómo | Esperado |
|---|---|---|---|
| 16 | El correo llega bien | Abrir el correo de la prueba 7 | Remitente `Equipo Seguridad LogiGho`, logo, código de 6 dígitos, enlace `?solicitud=...`, 15 minutos, "si no fuiste tú, ignora este correo", sin errores de formato |
| 17 | Confirmar con código incorrecto | `POST .../confirmaciones` con otro código | 400 `CODIGO_INVALIDO` y `intentos` = 1 en base |
| 18 | Contraseña débil | código correcto + `corta` | 400 `POLITICA_CONTRASENA` y **no** sube `intentos` |
| 19 | Confirmación correcta | código del correo + `Segura#2026x` | 201 `ACTIVO`; en `Users` roles `["Red social"]`, tiendas vacías; `RegisterContact` con `estado "Registrado en plataforma"` y `userId` (sin duplicar) |
| 20 | Código ya usado | repetir la 19 | 409 `SOLICITUD_YA_USADA` |
| 21 | Login del usuario nuevo | `InitiateAuth` USER_PASSWORD_AUTH con el correo | `id_token`; `GET /users` con ese token = 200 |
| 22 | `Red social` no administra nada | `GET /autenticacion/roles-asignables` con el token del usuario nuevo | 403 `PROHIBIDO` |
| 23 | Reenvío | `POST .../reenvio` antes de 60 s; luego después | 429 `DEMASIADAS_SOLICITUDES`; luego 202 y el código viejo da `CODIGO_INVALIDO` |
| 24 | Bloqueo por 5 intentos | 5 códigos malos y luego el correcto | el 6.º responde 429 `DEMASIADOS_INTENTOS` |
| 25 | Vencimiento | confirmar pasados 15 minutos | 410 `SOLICITUD_VENCIDA` |
| 26 | Dos confirmaciones a la vez | dos peticiones simultáneas con el código correcto | una responde 201 y la otra 409; un solo usuario creado |
| 27 | Roles asignables de Comercial | `GET /autenticacion/roles-asignables` con token de Comercial | 200 con `Tienda, Inversionista, Proveedor, Dropshipping` |
| 28 | Comercial asigna acceso | `POST .../usuarios/{userId}/acceso` con `{"roles":["Tienda"]}` al usuario de la 19 | 200; el usuario queda solo con `Tienda` (se quita `Red social`); fila en `LogAuditoria` |
| 29 | Rol que Comercial no puede dar | `{"roles":["Administrador"]}` | 403 `ROL_NO_PERMITIDO` con `Motivos` |
| 30 | Comercial no toca a un administrador | `usuarioId` de un `Administrador` | 403 `USUARIO_PROTEGIDO` |
| 31 | Nadie se asigna a sí mismo | `usuarioId` propio | 403 `AUTO_ASIGNACION_NO_PERMITIDA` |
| 32 | Pedir código de cambio de contraseña | `POST /cambio-pass/codigo` con token | 200 con `solicitudId`; llega el correo; otra petición en menos de 60 s = 429 |
| 33 | Cambiar contraseña con código | `POST /cambio-pass` con actual, nueva, código y `solicitudId` | 200; el login con la contraseña vieja falla y con la nueva funciona |
| 34 | Código de cambio incorrecto o ajeno | código malo / `solicitudId` de otra cuenta | 400 `CODIGO_INVALIDO` / 410 `CODIGO_VENCIDO`; la contraseña no cambia |
| 35 | Sin código | `POST /cambio-pass` sin `Codigo` | 400 `DATOS_INVALIDOS` |
| 36 | Token anterior tras el cambio (M-215) | usar el `id_token` previo al cambio | Documentar el resultado: hoy se acepta hasta vencer |

### 7.3 Antes de producción

- [ ] Pruebas 16 a 36 en verde y registradas aquí.
- [ ] Front construido sobre estos endpoints y probado con ellos en PreProd.
- [ ] SES fuera del sandbox en producción y DMARC en modo vigilancia (opcional, sin buzón de reportes).
- [ ] Rol `Red social` verificado en producción (ya existe), `ROLES_ASIGNABLES_COMERCIAL` y demás variables en la Library de producción.
- [ ] Índices de `RegistroSolicitudes` creados en DocumentDB (comprobar TTL y único, criterio CA21 de la spec).
- [ ] Iker da la instrucción explícita de producción; en producción cada paso lleva su visto bueno.
