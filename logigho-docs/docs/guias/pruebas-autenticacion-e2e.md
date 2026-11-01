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

## 7. Registro por OTP (cuando exista el backend)

La colección crecerá con la carpeta *6. Registro OTP*. Para probar el código sin leer el correo de verdad, el entorno de PreProd usa SES en modo *sandbox*: solo llega a correos verificados. Plan detallado en la spec `008_registro_autoservicio`.
