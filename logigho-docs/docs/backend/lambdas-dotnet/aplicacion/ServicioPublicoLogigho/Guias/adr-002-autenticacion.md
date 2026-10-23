---
autor: Iker Acevedo
fecha_creacion: 2026-10-22
estado: aceptada
---

# ADR-002 — Autenticación: OAuth2 client credentials + API key + perfil ApiClientes

**Fecha:** 2026-10-22
**Estado:** Aceptada

---

## Contexto

- `ApiIntegracionesExternas` (`rl7zt6zak5`) usa el pool Cognito `us-east-1_6jmSkfK4B` (3 usuarios). `POST /token` hace login con usuario y contraseña (`USER_PASSWORD_AUTH`) vía `APILambdaObtenerToken`; no hay `client_credentials`, scopes, WAF ni usage plans.
- El authorizer valida el header `Authorization`, pero `ApiLambaConsultarPedidos` lee primero el header `Token` (`Function.cs:51-57`) y solo decodifica el JWT sin verificar firma (`JwtClaimsHelper.cs:17-30`).
- Se requiere: muy seguro, bloqueo inmediato, permisos por tienda/producto/transportadora, trazabilidad.

## Opciones

| Opción | Resumen | Problema |
| --- | --- | --- |
| Usuario y contraseña Cognito (actual) | Contraseña humana en sistemas de terceros | Débil; sin scopes; sin refresh |
| Solo API key | Simple | Llave de larga vida; API key no es autenticación |
| **Client credentials + API key + perfil en Mongo** | Estándar máquina a máquina | Un header más para el integrador |

## Decisión

1. **Cognito `client_credentials`:** un app client con secreto por integrador en el pool `6jmSkfK4B`; resource server con scopes `guias:crear`, `guias:leer`, `guias:anular`, `cupo:leer`; access token corto. El authorizer exige el scope de cada método.
2. **API key + usage plan por cliente:** límite y cuota por cliente en el gateway; desactivar la key bloquea de inmediato.
3. **Perfil `ApiClientes`:** estado (`ACTIVO`/`BLOQUEADO`/`SUSPENDIDO_CUPO`), tiendas, productos, transportadoras, límites, IPs permitidas. Se consulta en cada request con caché de 30 s.
4. **WAF regional:** reglas administradas, límite por IP, lista de IPs bloqueadas.
5. Las tiendas y permisos salen **solo** del perfil en servidor.

`ApiClientes` unifica también a los usuarios de `/pedidos` (absorbe `ApiUsuariosTiendas` de la rama `feature/mejora-api-pedidos`). `/pedidos` y sus 3 usuarios siguen igual hasta su migración.

## Consecuencias

- Bloqueo: cambiar estado en `ApiClientes` (≤ 30 s) + desactivar API key (inmediato) + IP en WAF si aplica.
- Rotación de secreto: nuevo app client, periodo de gracia, baja del anterior.
- Pendiente aparte: en `/pedidos`, leer solo `Authorization` o los claims del authorizer.
