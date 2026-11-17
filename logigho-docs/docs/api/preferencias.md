# API de Preferencias de usuario (`/preferencias/mia`)

**Para quién es:** el equipo de frontend que construye el [Sistema de Experiencias](../frontend/experiencias/index.md).
Todo lo que necesita para consumir la API está aquí y en el [contrato OpenAPI](preferencias.openapi.yaml).

**Qué hace:** guarda en el servidor la experiencia visual que eligió cada usuario y le dice cuáles puede usar.
La decisión de **qué experiencias existe y quién las ve la toma el servidor**, nunca el front.

| Entorno | URL base | Estado |
|---|---|---|
| PreProd | `https://8u5isw5u30.execute-api.us-east-1.amazonaws.com/PreProd` | Disponible cuando el backend se despliega y valida en PreProd (ver [estado](#estado-por-entorno)) |
| Producción | `https://7elgyten88.execute-api.us-east-1.amazonaws.com/Produccion` | **Todavía no.** Se monta cuando el front esté listo y Iker lo ordene |

Una sola ruta: `/preferencias/mia` (la "mia" es el propio usuario: **no existen identificadores en la ruta ni en el cuerpo**).

---

!!! danger "No escribir estas colecciones por otra vía"
    `PreferenciasUsuario` y `Experiencias` se escriben **solo** con esta API y la de [administración](experiencias.md). **No** usen `/metodoGenerico`, `insertarGenerico` ni scripts propios: se saltan las reglas de negocio.

## Autenticación

- Header `Token: <id_token de Cognito>` (el mismo que usan las demás APIs de la plataforma). **Use siempre `Token`**: el authorizer del gateway solo lee ese header (con `Authorization` el gateway responde 401 aunque la Lambda lo acepte).
- El gateway valida el token con el authorizer de Cognito y **la Lambda vuelve a verificar la firma** (defensa en profundidad).
- La identidad es **siempre el `sub` del token**. Enviar otro usuario en la query, un header o el cuerpo no tiene ningún efecto.
- No envíe nunca roles ni permisos: el servidor los lee de `Users`.

## `GET /preferencias/mia`

Devuelve la preferencia del usuario y las experiencias que puede elegir. **No escribe nada** (un usuario nuevo no tiene documento hasta su primer `PUT`).

```http
GET /PreProd/preferencias/mia HTTP/1.1
Token: eyJraWQiOi...
```

`200 OK`
```json
{
  "experiencia": "clasica",
  "tema": "claro",
  "version": 3,
  "disponibles": [
    { "id": "clasica", "nombre": "Clásica", "estado": "general", "orden": 10, "temas": ["claro"], "temaPorDefecto": "claro" },
    { "id": "prueba-interna", "nombre": "Prueba interna", "estado": "borrador", "orden": 100, "temas": ["claro"], "temaPorDefecto": "claro" }
  ]
}
```

| Campo | Tipo | Significado |
|---|---|---|
| `experiencia` | `string` | Experiencia activa del usuario. Si la guardada ya no está disponible (retirada, dejó de ser beta, perdió el rol) llega `clasica` |
| `tema` | `string` | Tema activo. Hoy solo `claro` |
| `version` | `number` | Versión del documento. **Se devuelve tal cual en el siguiente `PUT`**. `0` = el usuario nunca ha guardado |
| `disponibles` | `{ id, nombre, estado, orden, temas, temaPorDefecto }[]` | Lo que **este usuario** puede elegir, ordenado por `orden` (menor primero). Siempre incluye `clasica`. `estado`: `general`, `beta` o `borrador`. `temas` son los temas que soporta **esa** experiencia |

Usuario sin documento: `{"experiencia":"clasica","tema":"claro","version":0,"disponibles":[...]}`.

## `PUT /preferencias/mia`

```http
PUT /PreProd/preferencias/mia HTTP/1.1
Token: eyJraWQiOi...
Content-Type: application/json

{ "experiencia": "prueba-interna", "tema": "claro", "version": 3 }
```

Reglas del cuerpo (máximo **2048 bytes**): exactamente los tres campos, sin extras ni repetidos. `experiencia` y `tema` cumplen
`^[a-z][a-z0-9-]{1,39}$`, y el `tema` debe ser uno de los `temas` **de esa experiencia** (si no, `422 TEMA_NO_DISPONIBLE`). `version` es un entero entre 0 y 1.000.000.

`200 OK`: mismo formato que el `GET`, con `version` incrementada en 1 (el cuerpo trae el estado nuevo; no hace falta otro `GET`).

### Errores

Todo error trae `{ "codigo": "...", "traceId": "<12 hex>" }`. El `traceId` sirve para pedir ayuda al backend; no se muestra al usuario final.

| HTTP | `codigo` | Cuándo | Qué hace el front |
|---|---|---|---|
| 400 | `VALIDACION` | JSON inválido, campo faltante/extra/repetido, tipo o formato incorrecto | Es un bug del front: registrar y seguir con la experiencia actual |
| 401 | `SIN_TOKEN` · `TOKEN_INVALIDO` · `USUARIO_NO_ENCONTRADO` | Sin sesión válida (el gateway también responde 401) | Flujo normal de sesión vencida; mientras tanto usar `clasica` |
| 404 | `RUTA_NO_ENCONTRADA` | Ruta distinta de `/preferencias/mia` | Bug del front |
| 405 | `METODO_NO_PERMITIDO` | Método distinto de GET/PUT/OPTIONS | Bug del front |
| 409 | `CONFLICTO_VERSION` | Otro dispositivo guardó antes (la `version` enviada no es la actual) | Hacer `GET`, aplicar lo que llegó y **avisar al usuario**; no reintentar en bucle |
| 413 | `CUERPO_DEMASIADO_GRANDE` | Más de 2048 bytes | Bug del front |
| 422 | `EXPERIENCIA_NO_DISPONIBLE` | Id desconocido o no disponible para este usuario | Hacer `GET`, refrescar el selector y volver a `clasica` si hace falta |
| 422 | `TEMA_NO_DISPONIBLE` | Tema desconocido | Igual que el anterior |
| 500 | `ERROR_INTERNO` | Fallo del servidor | Usar `clasica`; reintentar más tarde |

> Un `401` generado por el **gateway** (token vencido o ausente) llega como `{"message":"Unauthorized"}` y, como todo error
> del gateway en esta plataforma, **puede no traer cabeceras CORS**: el navegador lo mostrará como error de red. El front
> debe tratar *cualquier* fallo como "usar `clasica`" y dejar que el interceptor de sesión haga lo suyo.

## Reglas de disponibilidad

| Estado de la experiencia | La ve |
|---|---|
| `general` | Todos los usuarios |
| `beta` | Usuarios con el rol **`Experiencia`**, los `Desarrollador` y los roles extra de la experiencia (`rolesAcceso`) |
| `borrador` | Los `Desarrollador` y los roles extra de la experiencia (`rolesAcceso`) |
| `retirada` | Nadie (no aparece en `disponibles`) |

Los estados y la configuración de cada experiencia los administra el equipo de desarrollo con la [API de administración](experiencias.md) (sin tocar la base de datos). Cuando negocio apruebe una experiencia, se cambia su estado de `beta` a `general` y **todos los usuarios** la ven en
`disponibles`; el front sigue mostrando a cada usuario **solo los módulos que ya tiene asignados** (esa parte no cambia).
`clasica` siempre existe y es `general`.

## Cómo consumirla (receta para el front)

1. **Arranque sin esperar a la red:** leer el id guardado en `localStorage` (solo el id, nada más), aplicar esa experiencia si está en el registro del front y, si no, `clasica`. Aplicar tokens de inmediato (sin parpadeo).
2. **Después del login:** `GET /preferencias/mia` con timeout corto (**5 s**). Si falla por cualquier motivo, seguir con lo que ya está aplicado.
3. **Validar lo que llega** contra el registro del front: un `experiencia` o `id` que el front no conoce se ignora (se usa `clasica`). El servidor es la fuente de verdad de *disponibilidad*; el registro del front lo es de *qué sabe pintar*.
4. **Selector:** mostrar solo los `disponibles` que el front conoce. Si solo hay una, ocultar el selector.
5. **Al cambiar:** aplicar la experiencia de inmediato (optimista), guardar el id en `localStorage` y hacer `PUT` con la `version` del último `GET`/`PUT`. Con la respuesta `200`, guardar la nueva `version`.
6. **Concurrencia:** `409` → `GET`, aplicar y avisar. `422` → `GET` y refrescar el selector. Nunca reintentar `409` en bucle.
7. **Sin polling:** un `GET` por sesión es suficiente.

```ts
// Tipos (espejo del contrato)
export interface ExperienciaDisponible {
  id: string; nombre: string; estado: 'general' | 'beta' | 'borrador';
  orden: number; temas: string[]; temaPorDefecto: string;
}
export interface PreferenciasUsuario {
  experiencia: string;
  tema: string;
  version: number;
  disponibles: ExperienciaDisponible[];
}
export interface ErrorPreferencias { codigo: string; traceId: string; }

// Adaptador mínimo (el puerto real lo define core/experiencias)
async function obtener(base: string, idToken: string): Promise<PreferenciasUsuario> {
  const r = await fetch(`${base}/preferencias/mia`, {
    headers: { Token: idToken },
    signal: AbortSignal.timeout(5000),
  });
  if (!r.ok) throw new Error(String(r.status)); // el llamador cae a 'clasica'
  return r.json();
}

async function guardar(base: string, idToken: string, cuerpo: { experiencia: string; tema: string; version: number }) {
  const r = await fetch(`${base}/preferencias/mia`, {
    method: 'PUT',
    headers: { Token: idToken, 'Content-Type': 'application/json' },
    body: JSON.stringify(cuerpo),
    signal: AbortSignal.timeout(5000),
  });
  if (r.status === 200) return (await r.json()) as PreferenciasUsuario;
  throw await r.json().catch(() => ({ codigo: `HTTP_${r.status}` })); // ErrorPreferencias
}
```

### Para construir sin backend (repositorio falso)

Respuestas de ejemplo para el repositorio falso de las pruebas:

```json
// GET, usuario sin rol y sin documento
{ "experiencia": "clasica", "tema": "claro", "version": 0, "disponibles": [ { "id": "clasica", "nombre": "Clásica", "estado": "general", "orden": 10, "temas": ["claro"], "temaPorDefecto": "claro" } ] }

// GET, usuario Desarrollador con una experiencia guardada
{ "experiencia": "prueba-interna", "tema": "claro", "version": 2,
  "disponibles": [ { "id": "clasica", "nombre": "Clásica", "estado": "general", "orden": 10, "temas": ["claro"], "temaPorDefecto": "claro" }, { "id": "prueba-interna", "nombre": "Prueba interna", "estado": "borrador", "orden": 100, "temas": ["claro"], "temaPorDefecto": "claro" } ] }

// PUT 409
{ "codigo": "CONFLICTO_VERSION", "traceId": "3fa85f6457a1" }

// PUT 422
{ "codigo": "EXPERIENCIA_NO_DISPONIBLE", "traceId": "9c1d2e3f4a5b" }
```

## CORS

El servidor concede CORS **solo** a los orígenes del front de cada entorno (lista cerrada; nunca `*`):

| Entorno | Orígenes |
|---|---|
| PreProd | `https://preprod-logigho.vercel.app` y `http://localhost:4200` |
| Producción | `https://logigho.com` (se confirma al montar producción) |

El preflight (`OPTIONS`) lo responde la propia Lambda (`204`, `Access-Control-Max-Age: 600`). Cabeceras permitidas: `Content-Type`, `Token`, `Authorization`.

## Límites y comportamiento operativo

- Throttling por método: `GET` y `OPTIONS` 50 peticiones/s (ráfaga 100); `PUT` 20/s (ráfaga 40). Pasado el límite el gateway responde 429 y el front usa `clasica`. Cada usuario necesita muy pocas llamadas (un `GET` por sesión y un `PUT` por cambio).
- Respuestas con `Cache-Control: no-store`.
- Metas de rendimiento (a medir en PreProd): Lambda en caliente p95 < 100 ms (`Duration`), arranque en frío < 500 ms (`Init Duration`, Native AOT); extremo a extremo (con el gateway) p95 < 300 ms.

## Estado por entorno

| Entorno | Backend | Colección | Última validación |
|---|---|---|---|
| PreProd | Código y documentación listos; **despliegue y validación en curso** (se actualiza esta tabla al terminar) | `PreferenciasUsuario` con índice único `ux_sub` (lo crea Iker) | Pendiente |
| Producción | No montado | No creada | — |

En PreProd se crea la experiencia de prueba `prueba-interna` (estado `borrador`, la ve solo el rol `Desarrollador`) con `POST /experiencias` para probar el ciclo completo. El catálogo está en la colección `Experiencias` y los cambios tardan hasta ~60 s en verse para todos.
Usuarios de prueba: los define Iker (uno sin rol, uno `Desarrollador`, opcionalmente uno con rol `Experiencia`).

## Documentos relacionados

- [API de administración de experiencias](experiencias.md) · [Contrato OpenAPI](preferencias.openapi.yaml) · [Ficha de la Lambda y operación](../backend/core/preferencias-usuario/index.md) · [ADR-001](../backend/core/preferencias-usuario/ADR-001-identidad-y-catalogo-del-servidor.md)
- [Reglas de cambio entre experiencias](../frontend/experiencias/reglas-de-cambio.md) · [Seguridad por diseño](../frontend/experiencias/seguridad-por-diseno.md) · [Plan de implementación (F4)](../frontend/experiencias/plan-implementacion.md)
