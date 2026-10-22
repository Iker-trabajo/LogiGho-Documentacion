# API de Inventario

**Autor:** Iker Acevedo · **Fecha de creación:** 2026-10-21 · **Estado:** contratos validados en PreProd; producción pendiente

Rutas del sistema [`ApiLambdaInventario`](../backend/inventario/index.md). **Todas viven en el REST único de cada entorno**:

| Entorno | API REST | Stage |
|---|---|---|
| Producción | `API_LogiGho_Rest` · `7elgyten88` | `Produccion` |
| PreProd | `API_LogiGho_Rest_PreProd` · `8u5isw5u30` | `PreProd` |

!!! warning "El puente `g3iz6qk3f0` es temporal"
    Hoy las rutas de inventario de PreProd se prueban por el HTTP API `g3iz6qk3f0` (`https://g3iz6qk3f0.execute-api.us-east-1.amazonaws.com`, sin stage). Es un atajo **solo de PreProd** y se retira cuando las rutas existan en el REST `8u5isw5u30` ([Tareas futuras](../backend/inventario/tareas-futuras.md#retirar-el-puente-g3iz6qk3f0)). Las URLs de este documento son relativas al REST del entorno. Nada de producción usa el puente.

## Resumen de rutas

| Método y ruta | Función | Authorizer en producción | Roles |
|---|---|---|---|
| `POST /actualizaInventarioLotes` | `ApiLambdaInventarioPorCarga` | ninguno (se conserva) | — (la llaman otras lambdas sin token) |
| `POST /actualizarInventario` | `ApiLambdaInventarioPorGuia` | Cognito (se conserva) | — |
| `GET /inventario/trazabilidad` | `ApiLambdaInventarioTrazabilidad` | ninguno; la lambda verifica el JWT | `ROLES_TRAZABILIDAD` |
| `GET/POST/PUT/DELETE /inventario/suscripciones…` | `ApiLambdaInventarioSuscripciones` | ninguno; verifica el JWT | dueño, o roles admin |
| `GET /inventario/despachos` | `ApiLambdaInventarioDespachos` | ninguno; verifica el JWT | `ROLES_DESPACHOS` |

Las dos primeras ya existen en producción y **solo cambian de lambda** (Strangler Fig): el contrato es el mismo. Las demás son nuevas. Las rutas de suscripciones y despachos pertenecen a las [alertas diarias](../backend/inventario/alertas-slack.md) y llegan a producción en su propia salida.

## Seguridad

- **Identidad.** Si la ruta tiene authorizer de Cognito, se usan sus claims. Si no, el front manda el `id_token` en la cabecera **`Token`** y la lambda **verifica la firma** contra el JWKS del user pool: algoritmo RS256 (nunca `none` ni HS256), emisor, vigencia (`exp`, `nbf`, 60 s de tolerancia), `token_use` y app client. Sin configuración rechaza todo.
- **Rol.** Con la identidad se busca el correo en la colección `Users` y se exige un rol de la lista de la ruta. Roles por defecto de trazabilidad y despachos: `Jefe Logistica`, `Administrador`, `Desarrollador`, `CEO`. Si la variable de roles queda vacía, **se niega a todos**. Cambiar roles es cambiar una variable de entorno (`ROLES_TRAZABILIDAD`, `ROLES_DESPACHOS`, `ROLES_ADMIN_SUSCRIPCIONES`), sin recompilar.
- **Rutas de escritura sin token.** `actualizaInventarioLotes` no tiene authorizer y el sistema nuevo conserva ese contrato porque la llaman lambdas servidor-a-servidor (por ejemplo DevolucionesMasivo, con la cabecera `headersecurity` y sin `Token`). Es un riesgo de plataforma ya conocido, no nuevo; se resuelve en el borde (authorizer o clave de API), no en la lambda.
- **Sin secretos en respuestas ni logs.** Los errores `500` no llevan detalle interno; el detalle va al log con el `correlationId`.

## CORS

!!! success "Confirmado: CORS abierto (`*`)"
    Por decisión de Iker del 2026-10-19 (riesgo aceptado), `CORS_ORIGENES = "*"` en las funciones de producción (Terraform `environments/prod/inventario`), igual que las lambdas gemelas. Con `*` la lambda **no** emite `Access-Control-Allow-Credentials`; la identidad viaja en la cabecera `Token`, no en cookies. Cada recurso REST lleva además su método `OPTIONS` (mock) para el preflight. Ver [ADR-INV-007](../arquitectura/adr/inventario/ADR-INV-007-cors-abierto.md).

!!! warning "Discrepancia documental"
    El README de la lambda aún dice que `*` se ignora y que hay lista blanca obligatoria. El código (`RespuestaHttp.cs`) sí acepta `*` y Terraform de producción lo configura. El E2E de PreProd comprobó el preflight del navegador (OPTIONS 204, POST 200, sin errores de CORS). Si algún día se cambia a lista blanca, hay que declarar los dominios reales del front, o el navegador descartará respuestas de movimientos que sí se guardaron.

## Formato común (rutas nuevas)

Respuesta de las rutas de lectura y de alertas:

```json
{ "exito": true, "datos": { }, "error": null, "correlationId": "…" }
```

Las dos rutas de escritura conservan el formato histórico (ver abajo).

---

## `POST /actualizaInventarioLotes` (por carga)

Registra movimientos de varias guías (por `Idcarga` o lista de `Guias`) y/o ingresos y ajustes manuales.

**Cuerpo** (nombres sin distinguir mayúsculas; `Idcarga`, `IdProducto` y las guías aceptan texto o número):

```json
{
  "Idcarga": "12345",
  "Estado": "Anulada",
  "Guias": ["240059153885"],
  "IgnorarMarcaProcesado": false,
  "IgnorarMarcaEntregasProcesado": false,
  "Ingresos": [ { "IdProducto": "10132", "Cantidad": 50 } ],
  "Ajustes":  [ { "IdProducto": "10132", "Cantidad": -3, "Fecha": "2026-10-21T15:00:00Z", "Motivo": "Conteo físico" } ]
}
```

| Campo | Notas |
|---|---|
| `Idcarga` | Lee los pedidos de esa carga por el campo indexado `Idcarga`, de 500 en 500 |
| `Guias` | Lee esas guías. **Sin `Idcarga` ni `Guias` no se lee ningún pedido** |
| `Estado` | Opcional; filtra los pedidos por estado (el front manda `"Anulada"`) |
| `IgnorarMarcaProcesado` | `true` = no filtrar por las banderas de procesado. Liquidaciones lo manda en `true` |
| `IgnorarMarcaEntregasProcesado` | `true` junto con el anterior = ignorar **todas** las banderas (para avisar un cambio de estado posterior a un cierre) |
| `Ingresos` / `Ajustes` | Acciones manuales, sin guía. **No son idempotentes**. Los ajustes aceptan `Fecha` (sin zona se lee como `America/Bogota`; se guarda en UTC), `Proovedor`, `Tienda` y `Motivo` opcionales |

**Respuesta 200** (formato de las gemelas):

```json
{ "Error": false, "Mensaje": null, "NumeroPaginas": 0, "TotalRegistros": 0, "Resultado": "Operación exitosa." }
```

Si no hubo nada que escribir, `Resultado` es `"Sin cambios: N pedidos leídos, M en Revisar"`.

!!! warning "No dependas del texto"
    Solo importa el código HTTP. El texto ha cambiado entre versiones y ningún llamador debe leerlo.

| Código | Cuándo |
|---|---|
| `200` | Procesado (incluye «sin cambios» y movimientos que ya existían) |
| `400` | Cuerpo vacío o inválido, o sin `Guia`/`Idcarga` ni `Ingresos`/`Ajustes` (`Error: true`, mensaje claro) |
| `500` | Error interno sin detalles |
| `503` / `504` | **Del API Gateway** si la llamada pasa de ~29–30 s. La lambda sigue trabajando y termina bien (ver [estándar de llamadas](../backend/inventario/llamadores.md#estandar-de-llamadas)) |

## `POST /actualizarInventario` (por guía)

Igual que la anterior para **una** guía.

```json
{ "Guia": "240059153885", "IgnorarMarcaEntregasProcesado": true, "Ingresos": [], "Ajustes": [] }
```

`Operacion` se ignora (nunca se usó). `IgnorarMarcaEntregasProcesado=true` equivale a ignorar todas las banderas. Respuesta y códigos: los mismos. En producción exige el authorizer Cognito (cabecera `Token` del usuario).

---

## `GET /inventario/trazabilidad`

Lista las guías de un producto uniendo las **5 colecciones de movimientos**, con su pedido y las marcas de descuadre.

`?idProducto=10402&limite=50&cursor=…&soloDescuadres=true`

| Parámetro | Notas |
|---|---|
| `idProducto` | Obligatorio |
| `limite` | 1–200 (por defecto 50) |
| `cursor` | Valor `siguienteCursor` de la página anterior. Paginación **siempre por cursor** |
| `soloDescuadres` | Solo las guías con alguna marca |

**Respuesta 200:**

```json
{
  "exito": true,
  "datos": {
    "guias": [
      {
        "guia": "240059153885",
        "pedidoEncontrado": true,
        "estadoPedido": "Devolución ratificada",
        "categoria": "RETORNO",
        "cantidadPedido": 3,
        "salida":     { "documentos": 1, "cantidad": 3 },
        "anulacion":  { "documentos": 0, "cantidad": 0 },
        "entrega":    { "documentos": 0, "cantidad": 0 },
        "devolucion": { "documentos": 1, "cantidad": 3 },
        "perdida":    { "documentos": 0, "cantidad": 0 },
        "marcas": ["RETORNO_PENDIENTE_BODEGA"]
      }
    ],
    "siguienteCursor": "10402|240059153999"
  },
  "error": null,
  "correlationId": "…"
}
```

Marcas (`marcas` vacío = la guía cuadra):

| Marca | Significa |
|---|---|
| `DUPLICADO` | Más documentos de un tipo que casillas del producto en el pedido |
| `CIERRE_DOBLE` | Más de un cierre a la vez |
| `SIN_SALIDA` | Hay otros movimientos pero ninguna salida |
| `CANTIDAD_DISTINTA` | Un movimiento no suma lo que dice el pedido |
| `CIERRE_FALTANTE` | El estado dice «cerró» y falta su movimiento |
| `TRANSITO_VENCIDO` | Más días en la transportadora que `DiasAlertaTransitoVencido` (30) |
| `SIN_PEDIDO` | Hay movimientos pero la guía no existe en `PedidosInter` |
| `RETORNO_PENDIENTE_BODEGA` | En retorno con una Devolución ya registrada; se conserva y espera «Devolución completada» |
| `ESTADO_CONTRADICTORIO` | En retorno pero con Entrega, Anulación o Pérdida registrada; se conserva y desarrollo revisa el estado |
| `ESTADO_DESCONOCIDO` | Estado sin homologar (categoría `REVISAR`); el sistema no lo toca |

| Código | Cuándo |
|---|---|
| `200` | OK |
| `400` | Falta `idProducto` o parámetro inválido |
| `401` | Sin identidad válida (token ausente, vencido o con firma inválida) |
| `403` | Usuario fuera de `Users` o sin rol permitido |
| `500` | Error interno sin detalles |

El front tiene un botón de Excel con el mismo contenido que la pantalla.

---

## `/inventario/suscripciones` (alertas diarias)

Autenticación: cabecera `Token`. **Cada usuario administra sus suscripciones** (por correo); `Jefe Logistica`, `Administrador`, `Desarrollador` y `CEO` pueden ver y editar las de todos con `?todas=true`. Una suscripción ajena responde `404` (no `403`, para no revelar que el id existe).

| Método y ruta | Qué hace | Éxito |
|---|---|---|
| `GET /inventario/suscripciones?todas=&cursor=&limite=` | Lista las tuyas. `limite` 1–100 (50); `datos.siguienteCursor` (null = última página) | 200 |
| `GET /inventario/suscripciones/{id}` | Una suscripción | 200 |
| `POST /inventario/suscripciones` | Crea (máximo 20 por usuario) | **201** |
| `PUT /inventario/suscripciones/{id}` | Edita; `webhook` opcional (si no viene, se conserva el guardado) | 200 |
| `DELETE /inventario/suscripciones/{id}` | Borra: `datos: { "eliminada": true }` | 200 |
| `POST /inventario/suscripciones/{id}/prueba` | Envía **ya** los mensajes (con `[PRUEBA]`) de una suscripción guardada; no toca `ultimoEnvio` | 200 |
| `POST /inventario/suscripciones/prueba` | Igual con un **borrador** (mismo cuerpo que el POST), sin guardar nada | 200 |

**Cuerpo de `POST`, `PUT` y borrador** (camelCase):

```json
{
  "nombre": "Reporte diario BTK",
  "activo": true,
  "webhook": "https://hooks.slack.com/services/<ids-del-webhook>",
  "horario": { "hora": "18:30", "zonaHoraria": "America/Bogota", "diasSemana": [1, 2, 3, 4, 5, 6] },
  "filtros": { "productos": [10402], "proveedores": [157041], "ecosistemas": ["BTK"] },
  "mensajes": { "despachosDelDia": true, "resumenInventario": true }
}
```

| Campo | Regla |
|---|---|
| `nombre` | Obligatorio, 1–80 caracteres |
| `webhook` | Obligatorio al crear y en el borrador; opcional al editar. Solo `https://hooks.slack.com/(services\|workflows\|triggers)/…` |
| `horario.hora` | `HH:mm` estricto |
| `horario.zonaHoraria` | Zona IANA; por defecto `America/Bogota` |
| `horario.diasSemana` | ISO 8601: 1 = lunes … 7 = domingo; al menos uno |
| `filtros` | Al menos **uno** de los tres con elementos (se unen). `proveedores` = **id numérico** de la tienda dueña (`Productos.IdTienda = Tienda.Id`), no el nombre. Máximos: 200 productos, 50 proveedores, 20 ecosistemas |
| `mensajes` | Por defecto los dos; al menos uno |

**Una suscripción** (`datos` de GET/POST/PUT; `datos.items[]` en la lista). El webhook completo **nunca vuelve**:

```json
{
  "id": "507f1f77bcf86cd799439011",
  "usuarioEmail": "usuario@ejemplo.com",
  "nombre": "Reporte diario BTK",
  "activo": true,
  "canal": { "tipo": "slack", "webhookUltimos4": "aB9z" },
  "horario": { "hora": "18:30", "zonaHoraria": "America/Bogota", "diasSemana": [1, 2, 3, 4, 5, 6] },
  "filtros": { "productos": [10402], "proveedores": [157041], "ecosistemas": ["BTK"] },
  "mensajes": { "despachosDelDia": true, "resumenInventario": true },
  "ultimoEnvio": { "fechaLocal": "2026-10-21", "estado": "OK", "detalle": null, "enviadoUtc": "2026-10-21T23:31:04.120Z" },
  "actualizadoUtc": "2026-10-21T20:10:00.000Z"
}
```

`ultimoEnvio` es `null` si nunca se envió; `estado` es `ENVIANDO`, `OK` o `ERROR`.

**Respuesta de prueba** (`datos`): `{ "ok": true, "mensajesEnviados": 2, "detalle": null }`. Si Slack rechaza: `{ "ok": false, "mensajesEnviados": 0, "detalle": "HTTP 404 (no_service)" }`.

!!! warning "Mira `datos.ok`, no solo el HTTP"
    La API responde `200` aunque Slack rechace el mensaje. El front debe leer **`datos.ok`**.

| Código | Cuándo |
|---|---|
| `400` | Validación (todos los errores juntos; **nunca repite el webhook**) |
| `401` | Sin identidad o sin correo en el token |
| `403` | Usuario fuera de `Users`, o sin rol admin al pedir `todas=true` |
| `404` | No existe o es ajena |
| `500` | Error interno sin detalles |

## `GET /inventario/despachos`

Guías y unidades por **día local**. Definición de «despachado»: [Alertas diarias](../backend/inventario/alertas-slack.md#que-es-despachado).

`?idProducto=10402,10408&desde=2026-10-20&hasta=2026-10-21&zona=America/Bogota`

| Parámetro | Regla |
|---|---|
| `idProducto` | Uno o varios separados por coma (hasta 100) |
| `desde`, `hasta` | Fechas **locales** `yyyy-MM-dd`, inclusivas, hasta 93 días |
| `zona` | Opcional; por defecto `America/Bogota` |

```json
{
  "exito": true,
  "datos": {
    "zona": "America/Bogota", "desde": "2026-10-20", "hasta": "2026-10-21", "productos": [10402],
    "dias": [ { "fecha": "2026-10-20", "guias": 310, "unidades": 905 }, { "fecha": "2026-10-21", "guias": 188, "unidades": 420 } ],
    "totales": { "guias": 498, "unidades": 1325 }
  },
  "error": null,
  "correlationId": "…"
}
```

Los días sin movimientos vienen en 0. La respuesta es acotada (un dato por día), sin paginación. Errores: los mismos que las suscripciones (`400`, `401`, `403`, `500`).

!!! note "Pendiente de decisión (D1)"
    Hoy la pantalla «Despachos por día» lee la API genérica y la ve quien ve el Resumen. Este endpoint exige los roles de la trazabilidad: un usuario con otro rol que hoy ve esa pantalla recibiría `403` hasta decidir si se abre más (`ROLES_DESPACHOS`).

## Invocaciones directas (sin API Gateway)

Para operación; las dispara EventBridge o una persona con permiso `lambda:InvokeFunction`.

| Función | Entrada | Respuesta |
|---|---|---|
| `BarridoCierre` | `{ "ModoSimulacion": true, "Productos": [10402], "Cursor": "…", "MaxGuias": 5000 }` (todo opcional; `ModoSimulacion` vale `true` si no se manda) | `Simulacion`, `GuiasRevisadas`, `Insertar`, `Corregir`, `Borrar`, `Aplicado`, `Muestra` (hasta 50 líneas), `SiguienteCursor`, `Completo`, `CierresConservadosEnRetorno` |
| `RecalcularResumen` | `{ "IdsRecalcular": [10402], "Cursor": 10402 }` (sin ids recorre los activos) | `{ "Recalculados": 65, "SiguienteCursor": null, "Completo": true }` |
| `ReporteDiario` | `{ "AhoraUtc": "2026-10-21T23:37:00Z" }` (solo para ensayos) | `Revisadas`, `Reclamadas`, `Enviadas`, `Fallidas`, `Omitidas`, `Completo` |
