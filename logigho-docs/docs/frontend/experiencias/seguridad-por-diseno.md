---

## Autor: Iker Acevedo

Fecha creacion: 2026-10-17
Estado: propuesta (pendiente de aprobación)
Tipo: estándar obligatorio

# Seguridad por diseño

Todo lo que se construya para el Sistema de Experiencias, y toda vista que se migre a él, cumple estas reglas.
Son obligatorias: un PR que no las cumpla no se aprueba. El modelo general de autorización está en
[Seguridad y accesos](../../arquitectura/seguridad-accesos.md).

## Principio

> **Una experiencia solo pinta.** La seguridad vive en el servidor. Ocultar un botón, un menú o una columna es
> experiencia de usuario, nunca un control de seguridad.

## Frontend

| # | Regla |
|---|---|
| F1 | Las versiones visuales y los shells no deciden permisos: los leen del facade, que a su vez los toma del backend (o, mientras tanto, de un único archivo de permisos de la vista) |
| F2 | Ningún dato que el usuario no pueda ver debe llegar al navegador. Si hay que ocultarlo, se pide al backend que no lo envíe |
| F3 | Nada de `innerHTML`, `bypassSecurityTrust*` ni HTML construido con texto del usuario o del API |
| F4 | En `localStorage` solo el id de la experiencia y el tema. Nunca tokens, roles, tiendas ni datos personales en código nuevo |
| F5 | Sin secretos en `environment.ts`: todo lo que está en el bundle es público |
| F6 | Sin scripts, estilos ni fuentes desde CDN o desde la base de datos en las experiencias |
| F7 | Las preferencias que llegan del servidor se validan contra el registro (id conocido y estado disponible) antes de usarse |
| F8 | Dependencias nuevas con versión fija y revisadas (`npm audit` en CI) |
| F9 | Los errores se muestran con mensajes genéricos; los detalles técnicos no se muestran al usuario |

## Backend (endpoints nuevos de este sistema)

| # | Regla |
|---|---|
| B1 | Toda ruta nueva va en el API REST con **authorizer Cognito**. Ninguna ruta nueva con autorización `NONE` |
| B2 | La identidad sale del `sub` del token validado. Nunca de un `email`, `Tienda` o `roles` enviados por el cliente |
| B3 | Endpoints específicos por caso de uso. **Prohibido** usar el CRUD genérico (`metodoGenerico`) para funcionalidades nuevas |
| B4 | Validación estricta de entrada: lista cerrada de campos (`UnmappedMemberHandling.Disallow`), tipos, longitudes y tamaños máximos |
| B5 | Rol IAM propio por Lambda con mínimo privilegio, como `ApiLambdaPublicarReporteAnalytics` |
| B6 | Logs sin tokens, headers completos ni datos personales: solo códigos de error e identificadores técnicos |
| B7 | CORS limitado a los dominios de la plataforma |
| B8 | Respuestas de error con códigos estables (`IDENTIDAD_AUSENTE`, `VALIDACION`), sin mensajes internos ni trazas |
| B9 | Pruebas de seguridad del caso de uso: sin token, token de otro usuario, entrada inválida, entrada demasiado grande |
| B10 | Infraestructura en Terraform; nada creado a mano en la consola |

## Revisión de PR

Todo PR del sistema incluye en su descripción:

- [ ] Cumple F1 a F9 (frontend) y B1 a B10 (si toca backend).
- [ ] No usa `metodoGenerico` en código nuevo.
- [ ] Las pruebas de seguridad del caso de uso pasan.

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-10-17 | Iker Acevedo | Creación de la propuesta |
