# Métricas — Seguridad

La lambda de métricas **no confía en un token por su contenido**: verifica su firma. Esta página explica cómo, por qué las rutas del gateway van sin authorizer y qué se corrigió en el camino.

## Quién puede qué

| Acción | Quién |
|---|---|
| Ver el resumen (`GET /metricas/entrega-ciudad`) | Cualquier usuario autenticado que exista en `Users` |
| Lanzar corridas, ver corridas, leer y guardar la configuración | Roles `CEO` y `Desarrollador` |

Sin token → **401** (`SIN_TOKEN`). Usuario válido sin el rol → **403** (`SIN_ROL_PERMITIDO`). Token con firma inválida, vencido o de otro emisor → **401**.

## Verificación de la firma del JWT

`VerificadorJwtCognito` (`Infraestructura/Seguridad`) es una copia adaptada del verificador de la lambda de inventario. Valida, **en este orden**:

1. **Algoritmo** `RS256` (nunca `none` ni `HS256`).
2. **Firma** contra la clave pública del JWKS del user pool, buscada por `kid`.
3. **Emisor** (`iss`) igual al del pool configurado.
4. **Vigencia** (`exp`, `nbf`) con un minuto de tolerancia de reloj.
5. **Tipo de token** (`token_use` = `id` o `access`).
6. **App client** (`aud` en el token de identidad, `client_id` en el de acceso) dentro de `COGNITO_CLIENT_IDS`.

Detalles de implementación:

- El JWKS se guarda en memoria por contenedor. Si llega un `kid` desconocido se vuelve a descargar, como máximo **una vez por minuto**, para que nadie pueda forzar descargas continuas.
- Un token ilegible o un fallo al leer el JWKS **nunca deja pasar** a nadie ni tumba la lambda: devuelve "sin identidad" (401). Solo se registran **códigos fijos** (`token-rechazado motivo=firma-invalida`), jamás el token ni sus claims.
- Variables: `COGNITO_REGION`, `COGNITO_USER_POOL_ID`, `COGNITO_CLIENT_IDS` (varias separadas por coma). **Si falta cualquiera**, `VerificadorNoConfigurado` rechaza todos los tokens (falla cerrado): Entregas y el panel responderían 401 a todo el mundo.
- El token se lee del encabezado `Token` (como lo manda el front) o de `Authorization: Bearer`.
- Solo después de verificar la firma se busca el `sub` en `Users` para decidir el rol.

## Por qué las rutas del gateway van con `authorization = NONE`

Las rutas REST de métricas se crean sin authorizer del gateway (igual que las de inventario, decisión de Iker 2026-10-09): **la lambda es la barrera**. Por eso el humo del pipeline espera **401** y no 403.

!!! warning "Discrepancia corregida (2026-10-29)"
    La primera versión de la lambda **solo decodificaba** el payload del JWT, con un comentario que decía que la firma la validaba "el authorizer del gateway". Pero Terraform crea las rutas con `authorization = NONE`: **nadie validaba la firma**. Cualquiera que conociera el `sub` de un CEO o Desarrollador podía fabricar un token y lanzar corridas o editar la configuración. Se corrigió con `VerificadorJwtCognito` antes del primer despliegue a producción (commit `5ef422d`, 201 tests en verde). La spec 004 (tarea 0.3) ya pedía "nunca solo decodificar sin validar".

## Qué NO hace esta seguridad

- Los conteos y porcentajes del resumen son visibles para cualquier usuario autenticado. No son datos personales (agregados por ciudad y transportadora).
- El [ajuste visual](ajuste-visual.md) no oculta el dato real: el backend entrega siempre el porcentaje real y el front lo ajusta al mostrarlo.
- El front oculta el menú de administración por rol, pero **no es la barrera**: el backend vuelve a validar el rol en cada llamada.

## Pruebas

- Tests unitarios del verificador: firma con otra clave, token sin firma, payload alterado después de firmar, `alg` `none`/`HS256`/`RS512`, firma vacía, vencido, `nbf` futuro, otro emisor, `token_use` inválido, otro app client, token malformado, JWKS caído, rotación de claves.
- Tests del enrutador: token con firma inválida → 401 aunque el `sub` sea de un admin; sin encabezado no se llama al verificador.
- Colección Postman en `specs/004_resumen_entrega_ciudad/pruebas/` con la prueba de "firma alterada → 401".
