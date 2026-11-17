---

## Autor: Yadilton Torres

Fecha creacion: 2026-11-11  
Estado: activo  
Tipo: guía

# PreProd: API HTTP y API REST

En **PreProd** hay dos API Gateway mientras backend migra todo a la REST. El frontend reparte las llamadas entre las dos. En **producción** no aplica: `apiUrl` y `apiUrlRest` son la misma URL.

| API | ID | Variable | Qué tiene |
| --- | --- | --- | --- |
| HTTP ("API REST TEST") | `g3iz6qk3f0` | `apiUrl` | La de siempre: `metodoGenerico`, `module`, `registerContact`, pedidos, guías, `getObject` / `putObject`, tiendas, etc. |
| REST (`API_LogiGho_Rest_PreProd`) | `8u5isw5u30`, stage `PreProd` | `apiUrlRest` | Réplica de producción con los métodos ya migrados: `autenticacion/*`, `createUser`, `metricas/*`, `users`, `novedades`, `inventario`, `preferencias`, `experiencias`, `webhooks` |

---

## Cómo decide el frontend

`src/app/core/application/services/consumo-generico/consumo-generico.service.ts`:

```ts
export const RUTAS_REST = ['autenticacion/', 'createUser', 'metricas/', 'users'];

export function baseApiPara(nombremetodo: string): string {
  // REST si la ruta empieza por una de RUTAS_REST; si no, la HTTP
}
```

Todos los métodos de `ConsumoGenericoService` (`consultarGenerico`, `insertarGenerico`, `actualizarGenerico`, `eliminarGenerico`, etc.) arman la URL con `baseApiPara`. Los servicios que arman su propia URL usan la variable directamente:

| Servicio | Ruta | Variable |
| --- | --- | --- |
| `register-service.service.ts` | `createUser` | `apiUrlRest` |
| `usuariotienda-service.service.ts` | `users` | `apiUrlRest` |
| `configuracion-entrega.service.ts`, `metricas-entrega.service.ts` | `metricas/entrega-ciudad` | `apiUrlRest` |

**Cuando backend migre una ruta** a la REST, se agrega a `RUTAS_REST`. **Cuando la REST tenga todo**, `apiUrl` de `environment.preprod.ts` vuelve a la REST y `RUTAS_REST` deja de importar.

---

## Auditoría en PreProd

`insertarAuditoria` no existe en ninguna de las dos API de PreProd. En `environment.preprod.ts` queda `medotoAuditoria: ''` y el frontend solo audita si tiene valor (`ConsumoGenericoService` y `AuthenticateService`). En producción sigue `insertarAuditoria`: **no cambiarlo**.

---

## Síntomas y causa

| Síntoma | Causa real |
| --- | --- |
| Consola llena de "blocked by CORS policy … No 'Access-Control-Allow-Origin'" contra `8u5isw5u30` | La ruta **no existe** en la REST. API Gateway REST responde `403 {"message":"Missing Authentication Token"}` sin encabezados de CORS y el navegador lo muestra como CORS. El CORS de la REST está bien (`*`, con `Token` y `headersecurity`) |
| La página carga sin datos (muro vacío, sin módulos, sin notificaciones) | Lo mismo: `metodoGenerico` o `module` apuntando a la REST |
| La sesión queda con los roles de otra cuenta; el Panel comercial saca al Muro; el formulario Accesos no sale | `/users` (en las dos API) no encuentra algunas cuentas que sí están en `Users`. El layout tiene respaldo por `metodoGenerico?coleccion=Users` (ver [Default Layout](../frontend/layout/default-layout.md#carga-del-usuario)) |

---

## Cómo comprobarlo (solo lectura)

```bash
curl -s -X OPTIONS "https://8u5isw5u30.execute-api.us-east-1.amazonaws.com/PreProd/<ruta>" -H "Origin: http://localhost:4200" -H "Access-Control-Request-Method: GET"
```

- `{"message":"Missing Authentication Token"}` → la ruta no existe en la REST.
- `200` con `Access-Control-Allow-Origin: *` → existe y el CORS está bien.

En la HTTP, `OPTIONS` siempre responde 204; para saber si una ruta existe se usa `GET`: `{"message":"Not Found"}` = no existe (o solo acepta `POST`).

A qué API va cada llamada, desde la consola del navegador:

```js
performance.getEntriesByType('resource').filter(e => /execute-api/.test(e.name)).map(e => e.name.split('?')[0])
```

`g3iz6qk3f0` = HTTP, `8u5isw5u30` = REST, `7elgyten88` = producción.

---

## Servidor local

`ng serve` usa la configuración `preprod` por defecto. Para producción: `ng serve --configuration production`. Si `npm ci` falla con `EPERM … esbuild.exe`, hay un servidor corriendo: detenerlo primero.

---

## Pendiente de backend

- Crear en la REST las rutas que faltan (`metodoGenerico`, `module`, `registerContact`, `insertarAuditoria`).
- Revisar `/users` de PreProd: no encuentra usuarios que sí existen en `Users`.
- La HTTP devuelve datos de `metodoGenerico` incluso sin sesión.

---

## Historial de cambios

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-11-11 | Yadilton Torres | Reparto de rutas entre HTTP y REST en PreProd (`RUTAS_REST`), auditoría apagada en PreProd y respaldo de `/users` |
