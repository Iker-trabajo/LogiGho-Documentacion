## Autor:
Fecha creacion: 2026-09-20
Estado: aceptada

# ADR-001 — Autenticación: decodificar el sub del JWT sin verificar firma, autorizar por rol en Users

**Autor:** Iker Acevedo
**Fecha:** 2026-09-20
**Estado:** Aceptada

---

## Contexto

El diseño inicial de esta lambda (antes de la primera implementación real) proponía un
modelo de autenticación propio para el dominio Datos: un `IValidadorJwtService` que
verificara la firma del JWT contra las claves públicas (JWKS) del User Pool de Cognito,
más una colección `IntegracionesExternas` para dar de alta y revocar credenciales de
integraciones (por ejemplo, del área de datos consumiendo la API sin pasar por Angular).

Al revisar ese diseño contra cómo funciona **realmente** el resto de la plataforma, dos
hechos cambiaron la decisión:

1. El API Gateway central (`g3iz6qk3f0`) **no tiene ningún Authorizer nativo en ninguna
   ruta existente** — se confirmó con `aws apigatewayv2 get-routes`, todas las rutas de
   todos los dominios usan `AuthorizationType: NONE`. La plataforma nunca dependió de
   que el Gateway validara el JWT.
2. Cada lambda nueva ya resuelve la identidad de la misma forma: decodifica el `sub` (o
   el `email`) del JWT **sin verificar la firma**, y confía en que esa firma ya fue
   validada antes en algún punto anterior del flujo (login contra Cognito). Es el mismo
   patrón que `JwtClaimsHelper` en `ApiLambdaConsultarPedidos` y `LectorTokenCognito` en
   `ApiLambdaDevolucionesMasivo`.

Construir un modelo de seguridad distinto (verificación de firma + colección de
integraciones propia) para un solo dominio nuevo introduciría una inconsistencia real:
esta API sería la única de toda la plataforma con un nivel de garantía criptográfica
distinto al resto, sin que ese nivel extra esté respaldado por ningún Authorizer en el
Gateway que lo haga cumplir de verdad.

---

## Opciones consideradas

### Opción A — Verificar firma JWT contra JWKS de Cognito + colección `IntegracionesExternas`

El diseño original: validar la firma del token contra las claves públicas del User Pool,
y mantener una colección propia de integraciones externas autorizadas (con estado
ACTIVA/INACTIVA como kill-switch).

**Pros:** garantía criptográfica real de que el token no fue falsificado, sin depender
de que nada anterior en la cadena ya lo haya validado. Kill-switch independiente del
resto de la plataforma.
**Contras:** requiere dependencias adicionales (`Microsoft.IdentityModel.Protocols.OpenIdConnect`,
`System.IdentityModel.Tokens.Jwt`), llamadas de red a JWKS (latencia + punto de fallo
nuevo), y una colección/tabla de permisos que no existe en ningún otro dominio de la
plataforma — sería el único lugar con ese modelo, aumentando la superficie que hay que
entender y mantener para alguien que ya conoce cómo funciona el resto del sistema.

### Opción B — Sub sin verificar firma + rol en `Users` (la que ya usa el resto de la plataforma)

Decodificar el payload del JWT para extraer `sub`, sin verificar nada de la firma.
Autorizar consultando la colección `Users` (la misma de toda la plataforma) por
`cognitoId == sub`, exigiendo que el documento tenga uno de los roles de negocio
permitidos (`Jefe Datos`, `Desarrollador`, `CEO`).

**Pros:** consistente con cómo ya funciona el 100% del resto de lambdas de la
plataforma. Cero dependencias nuevas de JWT/JWKS. Bloquear a alguien es la misma
operación que ya usa cualquier administrador (quitarle el rol en `Users`), sin una
pantalla ni un proceso nuevo que aprender. Reutiliza la tabla de roles que el equipo de
producto ya administra activamente.
**Contras:** no hay garantía criptográfica de que el `sub` decodificado sea auténtico —
un token adulterado con un `sub` de otro usuario pasaría la decodificación igual. Esto
es aceptable porque **ningún otro endpoint de la plataforma ofrece esa garantía
tampoco**: no se está bajando el nivel de seguridad de este dominio respecto al resto,
se está igualando.

---

## Decisión

**Se eligió:** Opción B.

**Razón:** la Opción A resuelve un problema de seguridad que la plataforma, como un
todo, no tiene resuelto en ningún otro lado — construirlo solo para este dominio da una
falsa sensación de garantía adicional sin cerrar el vector real (cualquier otro endpoint
seguiría aceptando un `sub` no verificado). Mejor ser consistente con el estándar real
de la plataforma hoy, y si algún día se decide agregar verificación de firma, que sea
una decisión transversal con un Authorizer en el Gateway, no un parche por dominio.

---

## Consecuencias

**Positivas:** cero infraestructura nueva de autenticación (sin JWKS, sin colección de
integraciones, sin kill-switch propio). Administrar acceso a esta API es una operación
que el equipo ya conoce: editar el arreglo de roles del usuario en `Users`. El código de
autorización (`AutorizacionUsuario`) es una única función de ~10 líneas, fácil de
auditar.

**Negativas:** la seguridad de este endpoint depende por completo de que ningún actor
pueda producir un JWT con un `sub` arbitrario sin haber pasado por un login real de
Cognito — dependencia que ya existe hoy en el resto de la plataforma, así que no es una
debilidad nueva introducida por este dominio, pero tampoco se resuelve acá. Si en el
futuro se decide agregar un Authorizer nativo al Gateway central, esta lambda no
necesita ningún cambio: seguiría leyendo el mismo header, ahora con la garantía de que
el Gateway ya lo validó antes de invocar.

---

## Impacto en el código

| Módulo / Repo | Cambio |
| ------------- | ------ |
| `Infraestructura/Servicios/DecodificadorJwt.cs` | Nuevo — reemplaza al `IValidadorJwtService`/`ValidadorJwtCognitoService` del diseño original (eliminados) |
| `Aplicacion/CasosUso/AutorizacionUsuario.cs` | Nuevo — reemplaza a `AutorizacionIntegracion.cs` (eliminado) |
| `Infraestructura/Repositorio/UsuarioRepository.cs` + `IUsuarioRepository.cs` | Nuevo — reemplaza a `IntegracionExternaRepository.cs`/`IIntegracionExternaRepository.cs` (eliminados) |
| `Dominio/Entidades/IntegracionExterna.cs` | Eliminado — no existe kill-switch propio de este dominio |
| `ApiLambdaPublicarReporteAnalytics.csproj` | Removidos `Microsoft.IdentityModel.Protocols.OpenIdConnect` y `System.IdentityModel.Tokens.Jwt` |
| `.preprod-hub/infra/templates/datos-infra-preprod.yaml` | Sin parámetros de Cognito (User Pool, App Client) — no hacen falta |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-09-20 | Iker Acevedo | Decisión inicial y documento, tras revisar el diseño original contra el comportamiento real del Gateway (`aws apigatewayv2 get-routes`) y el patrón ya usado en `ApiLambdaConsultarPedidos`/`ApiLambdaDevolucionesMasivo`. |

---

## Referencias

- [ApiLambdaPublicarReporteAnalytics.md — Seguridad y autorización](../ApiLambdaPublicarReporteAnalytics.md#seguridad-y-autorización)
- [ApiLambdaDevolucionesMasivo.md — Seguridad](../../ApiLambdaDevolucionesMasivo/ApiLambdaDevolucionesMasivo.md#seguridad) — mismo patrón de decodificar sin verificar firma
