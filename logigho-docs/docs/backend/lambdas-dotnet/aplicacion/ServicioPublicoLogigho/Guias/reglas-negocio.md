---
autor: Iker Acevedo
fecha_creacion: 2026-10-23
ultima_actualizacion: 2026-10-23
estado: diseño
---

# Reglas de negocio — API pública de creación de guías

> **Estado:** reglas levantadas en entrevista con Iker el 2026-10-23, contrastadas con el código de `ApiLambdaCrearGuia`, `ApiLambdaCargarGuias`, `ApiLambdaAnularGuia` y el modal Angular `modal-creacion-pedidos`. No hay código de la API desplegado.
> Cupo financiero y alarmas de pedidos tienen su propio documento: [cupo.md](cupo.md) y `alarmas-pedidos.md` (en construcción). Aquí solo se referencian.
> **Pendiente a futuro:** Iker creará una base de conocimiento; migrar estas specs allí cuando exista.

---

## Cómo leer este documento

Cada regla tiene un id `RN-XXX`, un enunciado, su fuente y su estado.

| Estado | Significado |
| --- | --- |
| **Confirmada** | Iker la decidió o la aprobó (entrevista 2026-10-23, brief del proyecto o ADR/documento ya aprobado) |
| **Pendiente** | Falta un dato o una decisión. Hay una recomendación, pero no es regla hasta que Iker la confirme |

**Pendientes:** los ids `RN-P-xx` de este documento se llamaban `P-xx`; se renombraron para no chocar con los de [pendientes-preprod.md](pendientes-preprod.md) (decisiones antes de PreProd).

**Fuentes:** `CR` = `Lambdas_NET_LogiGho/LambdasLogiGho.Aplicacion/Logigho/ApiLambdaCrearGuia`; `CG` = `.../ApiLambdaCargarGuias/src/ApiLambdaCargarGuias`; `AN` = `.../ApiLambdaAnularGuia/src/ApiLambdaAnularGuia`; `Modal` = `Angular SitioLogiGho/SitioLogiGho/src/app/components/modal-creacion-pedidos/modal-creacion-pedidos.component.ts`; **"Decisión Iker"** = entrevista del 2026-10-23.

Los nombres de campo de la API son propuestos (camelCase); la columna "Campo actual" indica el campo del flujo de hoy en `CargaPedido`.

---

## 1. Alcance y principios

| Id | Regla | Fuente | Estado |
| --- | --- | --- | --- |
| RN-001 | La API replica el flujo actual y sus destinos: inserta en `CargaPedido`, llama a la transportadora, actualiza `CargaPedido`, inserta en `PedidosInter` (mismo formato PascalCase, `Estado = "Creado"`), dispara la etiqueta (`/creacionEtiqueta`, Step Function `Step_Creacion_Etiqueta`, buckets `bucket-guias-inter-prod` y `bucket-guias-pdf-prod`) y avisa al inventario (`/actualizaInventarioLotes {Guias}`) | `CR/.../CargaInterUseCase.cs:251-255`; `Modal:1343-1348`; `ApiPublicaGuias.md` | Confirmada |
| RN-002 | Transportadoras de v1: Interrapidísimo primero (F1); luego Envia, TCC, Servientrega y D2E (F3). XCargo (`D2`) no | Decisión Iker | Confirmada |
| RN-003 | Fuera de v1: contraentrega (COD), recogidas (`FECHA RECOGIDA`), cotización y operación fuera de Colombia. Por tanto `APLICA CONTRA PAGO` es siempre `NO` y `Total recaudo` es `0` | Decisión Iker; `ApiPublicaGuias.md` | Confirmada |
| RN-004 | Las reglas que varían por transportadora (valor declarado mínimo, límites de peso y dimensiones, longitudes de texto, capacidades de entrega y anulación) viven en una **colección de reglas en BD**, no en variables de entorno ni en el código. Agregar una transportadora o ajustar un límite no exige desplegar | Decisión Iker | Confirmada |
| RN-005 | Los defectos conocidos no se replican: guías duplicadas por reintentos, pedidos huérfanos, transportadora desconocida que cae a Envia, y cliente que manda `IdTienda`/`TiendaProveedor`/`Estado` arbitrarios | `hallazgos-lambdas-actuales.md` | Confirmada |

---

## 2. Campos de entrada de la guía

El body de `POST /v1/guias` es un objeto con los campos de esta sección. Un lote ([sección 7](#7-lotes)) es una lista de estos mismos objetos.

### 2.1 Tabla de campos

| Campo API | Campo actual | Obligatorio | Formato / valores | Regla |
| --- | --- | --- | --- | --- |
| `referencia` | — (nuevo) | Sí en lotes | Texto, único por cliente | RN-010 |
| `tiendaId` | `IdTienda` | Sí | Id de una tienda permitida al cliente | RN-030 |
| `transportadora` | `TRANSPORTADORA` | Sí | `INTERRAPIDISIMO`, `ENVIA`, `TCC`, `SERVIENTREGA`, `D2E` | RN-025 |
| `destinatario.nombres` | `NOMBRE COMPLETO` | Sí | Texto | RN-016 |
| `destinatario.apellidos` | `NOMBRE COMPLETO` | Sí | Texto | RN-016 |
| `destinatario.tipoDocumento` | fijo `"CC"` en Inter/TCC | No | `CC`, `CE`, `NIT`, `PAS`, `TI`; default `CC` | RN-017 |
| `destinatario.numeroDocumento` | `IDENTIFICACION (OPCIONAL)` | No | Texto numérico | RN-017 |
| `destinatario.telefono` | `TELEFONO` | Sí | Dígitos nacionales del país de la tienda (CO: 10 dígitos) | RN-018 |
| `destinatario.correo` | `CORREO (OPCIONAL)` | No | Correo válido | RN-021 |
| `destinatario.direccion` | `DIRECCION` | Sí | Texto, largo máximo por transportadora | RN-019 |
| `destinatario.ciudad` | `CIUDAD` | Sí | DANE de 8 dígitos | RN-050 |
| `peso` | `PESO` | Sí | kg, decimal, máx. 2 decimales | RN-013, RN-014 |
| `largo`, `ancho`, `alto` | `LARGO`, `ANCHO`, `ALTO` | Sí | cm | RN-013 |
| `valorDeclarado` | `VALOR DECLARADO` | Sí | Entero COP > 0 y ≥ mínimo de la transportadora | RN-011, RN-012 |
| `tipoEntrega` | `TIPO ENTREGA` | No | `1` entrega en dirección (default); `2` reclame en oficina | RN-053 |
| `diceContener` | `DICE CONTENER` | Por definir | Texto | RN-022 |
| `observaciones` | `OBSERVACIONES` | No | Texto | RN-022 |
| `productos[]` | `ID STOCK n` / `CANTIDAD STOCK n` | Según tienda | `{ idProducto, cantidad }`, máx. 18 | RN-035 a RN-037 |
| `tipoEtiqueta` | `TIPO ETIQUETA` | No | `Sticker` o `Mediana`; default `Sticker` | RN-020 |
| `datosAdicionales` | — (nuevo) | No | Objeto con claves permitidas por cliente | RN-024 |

### 2.2 Reglas

| Id | Regla | Fuente | Estado |
| --- | --- | --- | --- |
| RN-010 | En lotes cada guía trae `referencia` obligatoria (normalmente el id del pedido del integrador), única por cliente. Cumple el rol de idempotency key de esa guía. En `POST /v1/guias` la clave es el header `Idempotency-Key` ([ADR-003](adr-003-idempotencia.md)) | Decisión Iker; `ADR-003` | Confirmada |
| RN-011 | `valorDeclarado` es obligatorio y entero COP > 0. **No hay default silencioso**: hoy Inter cae a 10 000 si el valor no parsea (`CR/.../CargaInterUseCase.cs:92`) y Envia usa un `VALOR BASE PRODUCTO` fijo de 10 000 (`CR/.../CargaEnviaUseCase.cs:173`) | Decisión Iker | Confirmada |
| RN-012 | El valor declarado **mínimo se parametriza por transportadora** en la colección de reglas (RN-004). La configuración deja espacio para distinguir con/sin recaudo cuando entre COD. **Valor inicial (decisión de Iker, 2026-10-24): 45 000 COP para TODAS las transportadoras, con o sin recaudo.** Lo que distingue con y sin recaudo es el campo `AplicaContraPago` (en v1 siempre `NO`); se revisa si aparecen problemas | Decisión Iker | Confirmada |
| RN-013 | `peso`, `largo`, `ancho` y `alto` son obligatorios, sin default. Hoy los use cases parsean a 0 si el valor es inválido (`CR/.../CargaInterUseCase.cs:98-101`) y el front manda 1 kg y 10×10×10 cm por defecto (`Modal:1227-1230`). Valor ausente, no numérico o ≤ 0 → error de campo | Decisión Iker | Confirmada |
| RN-014 | El peso acepta decimal (máx. 2 decimales). Si la transportadora exige entero se **redondea hacia arriba**, y la respuesta devuelve el peso facturable ya redondeado. Hoy Envia/D2E hacen `int.TryParse` (un peso `1.5` queda en 0), Servientrega hace `Floor` (`CargaServientregaUseCase.cs:161`) e Inter/TCC aceptan decimal | Decisión Iker | Confirmada |
| RN-015 | Límites mínimos y máximos de peso y de dimensiones (lado máximo, suma de lados) por transportadora, configurables en la colección de reglas. Hoy el código no valida máximos (Inter siempre crea `PAQUETE PEQUEÑO`, `IdTipoEnvio = 3`; TCC pasa a paquetería si el peso > 5 kg). Mientras no haya valores, la API devuelve el error de la transportadora | Decisión Iker | Confirmada (valores: ver RN-P-02) |
| RN-016 | El destinatario se envía con `nombres` y `apellidos` separados, ambos obligatorios. La API arma el nombre completo internamente. Elimina el parseo frágil por espacios (con una sola palabra Inter duplica nombre y apellidos, `CR/.../CargaInterUseCase.cs:151-153`) | Decisión Iker | Confirmada |
| RN-017 | El documento es opcional. `tipoDocumento` es un enum (`CC`, `CE`, `NIT`, `PAS`, `TI`) con default `CC`; si viene número sin tipo se asume `CC`. Hoy el tipo está fijo en `"CC"` (`CR/.../CargaInterUseCase.cs:130`). El mapeo del tipo a lo que acepta cada transportadora se define al integrar cada una (ver RN-P-06) | Decisión Iker | Confirmada |
| RN-018 | El teléfono se valida y normaliza según el país de la tienda: acepta `+57`, espacios y guiones y guarda solo los dígitos nacionales. Colombia: celular de 10 dígitos (`3XXXXXXXXX`) o fijo de 7/10 dígitos. Inválido → error de campo, nunca `0` (hoy `long.TryParse` cae a `0`, `CR/.../CargaInterUseCase.cs:156`). Las reglas por país están en el catálogo de RN-101 | Decisión Iker | Confirmada |
| RN-019 | Los largos máximos de texto (dirección, observaciones, "dice contener", nombres) son **por transportadora** en la colección de reglas. Si se exceden, la API **rechaza** indicando el máximo; no trunca. Hoy TCC trunca en silencio (dirección 50, observaciones 250, nombre 60; `CargaTccUseCase.cs:124-152`) | Decisión Iker | Confirmada (valores de Inter: ver RN-P-02) |
| RN-020 | `tipoEtiqueta` es opcional, enum `Sticker` \| `Mediana`, default `Sticker`. La respuesta de creación no espera el PDF: la etiqueta se genera de forma asíncrona (RN-001) y se obtiene con `GET /v1/guias/{numero}/etiqueta` | Decisión Iker; `endpoints.md` | Confirmada |
| RN-021 | `correo` es opcional; si viene debe tener formato de correo válido | Recomendación | Pendiente |
| RN-022 | `diceContener` obligatorio (el flujo actual lo manda siempre y alimenta `Productos` en `PedidosInter`, `CR/.../CargaInterUseCase.cs:230`); `observaciones` opcional con default `Pedido generado desde Logigho`, que es el valor del front (`Modal:1239`) | Recomendación | Pendiente |
| RN-023 | Los siguientes campos los fija el servidor y el cliente **no puede enviarlos**: `Estado` (`Pendiente` al ingresar, luego el flujo), `Origen` (`API`), `IdCarga`, `Tienda` (nombre), `TiendaProveedor`, sucursal remitente, `CIUDAD ORIGEN`, `DIRECCION ORIGEN`, `Total Flete`, `Total Servicio`, `Total Sobreflete`, `Total recaudo`, `NumeroPreenvio`, `APLICA CONTRA PAGO` y `FORMA DE PAGO` (esta última según `reglas-modalidad-pago.ts`). Los defaults del front (`ASESOR`, `CODIGO OFICINA = 1`, `CODIGO REGIONAL = 3`, `SEGURO`, `TELEFONO REMITENTE`) se definen al implementar cada transportadora (ver RN-P-07) | Decisión Iker; `Modal:1210-1251` | Confirmada (valores fijos: ver RN-P-07) |
| RN-024 | Esquema **estricto**: un campo no definido o prohibido → `400 CAMPO_NO_PERMITIDO` con el nombre del campo. Excepción: `datosAdicionales`, un objeto cuyas claves permitidas, tipo y largo máximo se configuran **por cliente API**. Un cliente normal no tiene ninguna; las tiendas propias de LogiGho, que también usarán la API y el MCP, tienen sus 2–3 campos extra (datos de venta, etc.). Los valores se guardan en `CargaPedido`/`PedidosInter` con el mismo nombre que usan hoy | Decisión Iker | Confirmada (nombres de los campos: ver RN-P-03) |
| RN-025 | `transportadora` fuera del enum → `400`; no permitida al cliente → `403 TRANSPORTADORA_NO_PERMITIDA`. Nunca cae a Envia (hoy el `switch` usa Envia como default, `CR/Function.cs:151`, `CG/Function.cs:130-135`) | Decisión Iker; `hallazgos-lambdas-actuales.md` | Confirmada |

---

## 3. Tienda, origen, sucursal remitente y productos

| Id | Regla | Fuente | Estado |
| --- | --- | --- | --- |
| RN-030 | `tiendaId` es obligatorio y se valida contra las tiendas del cliente. Si no pertenece al cliente → `403 TIENDA_NO_PERMITIDA`. `TiendaProveedor` **nunca** se acepta del cliente | Decisión Iker | Confirmada |
| RN-031 | Una tienda puede pertenecer a varios clientes API; el permiso, los productos y la lista de transportadoras se evalúan por cliente | Decisión Iker | Confirmada |
| RN-032 | La **sucursal remitente** se resuelve en el servidor con la misma regla de hoy, nunca desde el body: si la guía tiene proveedor, se toma la `Sucursal` de la tienda proveedora; si no, la `Sucursal` de la tienda. Si queda vacía (o `N/A`) se usa la sucursal genérica. Solo Inter la usa (`CodigoConvenioRemitente` / `IdSucursal`) | `CR/Function.cs:114-121`; `CR/.../CargaInterUseCase.cs:103,123`; decisión Iker | Confirmada |
| RN-033 | **Origen del paquete** (ciudad y dirección): se toma de la configuración de la **tienda** (`CodigoDane`, `DireccionOrigen`). Si la tienda no la tiene, se usa la del **proveedor del producto**. Si ninguna existe → error. **Sin default a Medellín** (hoy el front cae a `Calle 43B # 82 - 62`, `Modal:1243-1244`). El integrador no envía origen | Decisión Iker | Confirmada (ver RN-P-04) |
| RN-034 | Todos los proveedores deben tener su origen configurado (acción de Iker antes de salir a producción) | Decisión Iker | Confirmada |
| RN-035 | **Tienda Dropshipping**: ≥ 1 producto obligatorio; todos del mismo proveedor (si no → error); cada producto debe existir y estar permitido para la tienda y para el cliente (públicos o privados asignados). `TiendaProveedor` y la sucursal se **derivan del producto en el servidor**. Replica el front (`Modal:1014-1058`, `1525-1650`) | Decisión Iker; `Modal` | Confirmada |
| RN-036 | **Tienda Propia**: los productos son opcionales; si vienen, deben existir y ser permitidos (el front lo valida también para tiendas propias, `Modal:443-449`) | Decisión Iker; `Modal` | Confirmada |
| RN-037 | `productos` es un arreglo `[{ idProducto, cantidad }]` con `cantidad` entero > 0, **máximo 18 líneas** (hoy `ID STOCK n`/`CANTIDAD STOCK n`: 12 en la creación individual y 18 en la masiva, `CR/.../CargaInterUseCase.cs:191`). Si hay ID debe haber cantidad y viceversa (`Modal:1525-1540`). El servidor calcula `Cantidad Total Productos` | Decisión Iker; `Modal` | Confirmada |
| RN-038 | La tienda se resuelve por **id**, no por nombre. Hoy la sucursal del proveedor se busca por `NombreTienda` en minúsculas (`CR/Function.cs:116`), lo que falla si hay nombres repetidos | `CR/Function.cs:114-121` | Confirmada |
| RN-039 | Tras crear la guía, la API avisa al inventario con `POST /actualizaInventarioLotes { Guias: [numero] }`, como hoy lo hace el front en la creación manual. Si el aviso falla, no se pierde: se reintenta desde la cola (RN-065) | `Modal:1343-1348`; `ADR-003` | Confirmada |

---

## 4. Ciudades, tipo de entrega, capacidades por transportadora

| Id | Regla | Fuente | Estado |
| --- | --- | --- | --- |
| RN-050 | `destinatario.ciudad` es código **DANE de 8 dígitos** (ej. `05001000`). Un DANE de 5 dígitos se normaliza a 8 (completa con `000`, como ya hace Envia, `CargaEnviaUseCase.cs:187-190`) | Decisión Iker | Confirmada |
| RN-051 | La API **valida la cobertura antes de llamar a la transportadora**. Ciudad sin cobertura → `422 CIUDAD_SIN_COBERTURA`. Catálogo de cobertura según transportadora: `CiudadesInter` (Inter, Envia, D2E), `CiudadesServientrega`, `CiudadesTCC` (solo `Estado = ACTIVA`, `CargaTccUseCase.cs:86-92`). Hoy Inter no valida que la ciudad exista (`CR/.../CargaInterUseCase.cs:94`) | `CR/Function.cs:69-78`; decisión Iker | Confirmada |
| RN-052 | La API expone un catálogo de ciudades con cobertura por transportadora. El nombre de ciudad y el departamento que se guardan en `PedidosInter` (`Ciudad`, `Departamento`) salen del catálogo, no del cliente | Decisión Iker; `CR/.../CargaInterUseCase.cs:249` | Confirmada |
| RN-053 | `tipoEntrega`: `1` = entrega en dirección (default), `2` = reclame en oficina. **Hoy solo Interrapidísimo ofrece reclame en oficina.** Con otra transportadora + `2` → error | Decisión Iker; `CR/.../CargaInterUseCase.cs:110-116` | Confirmada |
| RN-054 | Existe una **matriz de capacidades por transportadora** en la colección de reglas (RN-004): tipos de entrega soportados, anulación, límites de peso/dimensiones, largos de texto, valor declarado mínimo. Cuando se implemente otra transportadora con una capacidad nueva (ej. reclame en oficina) basta activarla en la matriz, sin cambiar el contrato | Decisión Iker | Confirmada |
| RN-055 | Por transportadora se mantienen las reglas propias que hoy viven en cada `Carga*UseCase` (ej. TCC: paquetería si peso > 5 kg y liquidación obligatoria antes del despacho con fail-fast, `CargaTccUseCase.cs:97,208-229`). La documentación detallada de cada una se agrega al integrar la transportadora (F3) | `CargaTccUseCase.cs` | Confirmada |

---

## 5. Errores de transportadora

| Id | Regla | Fuente | Estado |
| --- | --- | --- | --- |
| RN-060 | Los errores usan ProblemDetails (RFC 9457) con `codigo` estable y `traceId`; nunca stack ni mensaje interno. Catálogo de códigos en [endpoints.md](endpoints.md#errores) | `endpoints.md` | Confirmada |
| RN-061 | Si la transportadora rechaza la guía → `422 TRANSPORTADORA_RECHAZO` con `transportadora`, código y mensaje **originales** y un **mensaje normalizado** en español. Hoy el error vuelve como texto `"Error {mensaje}"` dentro de `Resultado[i]` con HTTP 200 y el front debe detectar `startsWith('ERROR')` (`Modal:1337-1338`). La API no usa ese formato | `endpoints.md`; `Modal:1337-1356` | Confirmada |
| RN-062 | Catálogo de mensajes normalizados: mapear los errores frecuentes de Inter a un código y un texto accionable para el integrador | Recomendación | Pendiente (RN-P-08) |
| RN-063 | Transportadora caída o circuito abierto → `503 TRANSPORTADORA_NO_DISPONIBLE`. Timeout o 5xx ambiguo tras enviar el POST → solicitud `INCIERTA`: **nunca se reintenta a ciegas el POST de creación**; solo se reintenta si el error ocurrió antes de enviar. Un reconciliador la resuelve y no se crea otra guía para esa solicitud hasta entonces | `ADR-003` | Confirmada |
| RN-064 | Compatibilidad con los destinos actuales: una guía rechazada deja en `CargaPedido` `Estado = "Rechazado"` y el mensaje en `ERROR`, como hoy (`CR/.../CargaInterUseCase.cs:264-270`). Estado de la solicitud en la API: `FALLIDA` | `CR/.../CargaInterUseCase.cs:264-270`; `ADR-003` | Confirmada |
| RN-065 | Los pasos posteriores a la creación (insert en `PedidosInter`, etiqueta, inventario) son idempotentes por número de guía; si fallan, la solicitud queda marcada y se reintenta desde la cola. **No se tragan** los errores como hoy (`CR/.../CargaInterUseCase.cs:257-260`), para no dejar pedidos huérfanos | `ADR-003` | Confirmada |

---

## 6. Anulación

`POST /v1/guias/{numero}/anulacion` (scope `guias:anular`).

| Id | Regla | Fuente | Estado |
| --- | --- | --- | --- |
| RN-070 | La anulación **solo marca la guía como `Anulada` internamente**, igual que `ApiLambdaAnularGuia` hoy (`AN/.../AnularGuiaUseCase.cs:64-67`). **No llama a la transportadora.** Riesgo conocido: la guía sigue viva en la transportadora (ver R-01) | Decisión Iker | Confirmada |
| RN-071 | La matriz de capacidades (RN-054) incluye `anulacionExterna` (`false` para todas en v1). Si más adelante una transportadora la soporta, se activa en la matriz | Decisión Iker | Confirmada |
| RN-072 | Solo se puede anular en **estados iniciales**, una lista configurable. El backend de hoy exige `Estado == "Creado"` (`AN/.../AnularGuiaUseCase.cs:52`); el front también admite "generada en…" y "en alistamiento del cliente" (`anular-guias.component.ts:267`, citado en `cupo.md`). Una guía despachada no se anula | Decisión Iker; `cupo.md` RC-024 | Confirmada (lista inicial: ver RN-P-05) |
| RN-073 | **Sin plazo en horas**: el estado ya limita la ventana | Decisión Iker | Confirmada |
| RN-074 | El cliente solo puede anular guías **creadas por él mediante la API** (`ApiSolicitudesGuia`), no cualquier guía de la tienda (ej. creadas por Angular o carga masiva) | Decisión Iker | Confirmada |
| RN-075 | Anular una guía ya anulada es idempotente: responde `200` con el estado actual (hoy `AN` responde error "ya se encuentra anulado", `AnularGuiaUseCase.cs:40-49`) | Decisión Iker | Confirmada |
| RN-076 | La anulación es gratis y libera la reserva de cupo por el monto vigente | `cupo.md` RC-024 | Confirmada |
| RN-077 | Efecto en inventario: se marca `Anulada` en `PedidosInter` y se avisa a `/actualizaInventarioLotes { Guias: [numero] }`. El reconciliador de `ApiLambdaInventario` ya reconoce `Estado = Anulada` y registra el movimiento `Anulacion`, que devuelve el stock (`CalculadoraResumen.cs`, `ClasificadorEstado.cs:26`) | `ApiLambdaInventario`; propuesta | Pendiente (RN-P-09) |

---

## 7. Lotes

`POST /v1/lotes` y `GET /v1/lotes/{id}`. Pensado para carga masiva.

| Id | Regla | Fuente | Estado |
| --- | --- | --- | --- |
| RN-080 | Máximo **50 guías por lote**, configurable en BD | Decisión Iker | Confirmada |
| RN-081 | **Lote parcial, no "todo o nada"**: se crean las guías válidas y las inválidas vuelven rechazadas en la misma respuesta. Una guía mala no tumba las otras 99. **Reemplaza** el borrador "valida todo antes" de `endpoints.md` | Decisión Iker | Confirmada |
| RN-082 | Respuesta `202`: `loteId` y, por guía, `{ indice, referencia, estado (ACEPTADA \| RECHAZADA), rechazoId, codigo, campo, mensaje }` | Decisión Iker | Confirmada |
| RN-083 | `rechazoId` es un identificador **único y no repetible** que genera la API por cada rechazo, para que el integrador y soporte lo citen | Decisión Iker | Confirmada |
| RN-084 | Una guía rechazada en la validación (campo inválido, ciudad sin cobertura, producto o tienda no permitidos, cupo) **no consume cupo, no crea registros ni consume la referencia** | Decisión Iker | Confirmada |
| RN-085 | Reuso de `referencia`: libre si la guía fue `RECHAZADA` o `FALLIDA`; bloqueada si está `CREADA` (devuelve el resultado guardado) o `INCIERTA` (`409` hasta que el reconciliador la resuelva) | Decisión Iker; `ADR-003` | Confirmada |
| RN-086 | Aceptado el lote, cada guía es independiente: una rechazada por la transportadora queda `FALLIDA`, libera su cupo y no afecta a las demás. Un mensaje SQS por guía (idempotente por id de solicitud) | `ADR-003`; `ApiPublicaGuias.md` | Confirmada |
| RN-087 | Cupo en un lote: se evalúa guía a guía en el orden del lote; las que ya no caben se rechazan con `CUPO_INSUFICIENTE` y las anteriores siguen aceptadas | Recomendación; definir con el chat de cupo | Pendiente (RN-P-10) |
| RN-088 | `GET /v1/lotes/{id}` devuelve el progreso y el estado final por guía (`CREADA`, `FALLIDA`, `INCIERTA`) con su número de guía | `endpoints.md` | Confirmada |

---

## 8. Clientes API

| Id | Regla | Fuente | Estado |
| --- | --- | --- | --- |
| RN-090 | Alta, edición, bloqueo/desbloqueo, rotación de secreto y permisos de clientes por `/admin/v1/clientes-api` con el pool interno. **Gerencia y Admin pueden todo**, con **motivo obligatorio** en cada cambio. Los cupos siguen la regla de [cupo.md](cupo.md) | Decisión Iker; `endpoints.md` | Confirmada |
| RN-091 | Todo cambio de gerencia o administradores queda en `ApiAuditoria` con antes, después, quién, cuándo y motivo | `ApiPublicaGuias.md`; decisión Iker | Confirmada |
| RN-092 | Cliente bloqueado o suspendido → `403 CLIENTE_BLOQUEADO`. El bloqueo cubre el perfil y la API key | `endpoints.md` | Confirmada |
| RN-093 | Permisos por cliente: tiendas, transportadoras, productos e IPs. Siempre salen del perfil en servidor, nunca del token ni del body | `ApiPublicaGuias.md` | Confirmada |
| RN-094 | **Productos**: el cliente hereda lo que la tienda puede usar hoy (públicos + privados asignados). El administrador puede definir una **lista explícita opcional** que restringe a esos productos. Una tienda en varios clientes puede tener listas distintas por cliente | Decisión Iker | Confirmada |
| RN-095 | **IP**: lista de IPs permitidas opcional por cliente (`403 IP_NO_PERMITIDA`); el WAF mantiene además el bloqueo global y el límite por IP | `ApiPublicaGuias.md`; `endpoints.md` | Confirmada |
| RN-096 | Las tiendas propias de LogiGho usan la misma API (y el MCP) como clientes API normales, con `datosAdicionales` (RN-024) | Decisión Iker | Confirmada |
| RN-097 | Qué pasa con las guías ya encoladas cuando un cliente se bloquea: ¿se terminan, se cancelan antes de llamar a la transportadora? | Recomendación: no llamar a la transportadora si el cliente quedó bloqueado antes de `CREANDO` | Pendiente (RN-P-11) |

---

## 9. Bases internacionales

La operación en otros países queda **fuera de v1**, pero el código y los datos no queman Colombia ni el peso colombiano.

| Id | Regla | Fuente | Estado |
| --- | --- | --- | --- |
| RN-100 | El **país** y la **moneda** los define la **tienda** (v1: `CO` / `COP`); el request no los envía. La guía guarda `codigoPais` (ISO 3166) y `moneda` (ISO 4217) y la respuesta los devuelve. Valor declarado, flete y cupo siempre en la moneda de la tienda, sin conversión | Decisión Iker | Confirmada |
| RN-101 | Las reglas que cambian por país (formato de ciudad, formato de teléfono, tipos de documento) viven en un **catálogo por país en BD**. v1 solo tiene las de Colombia: ciudad DANE8, teléfono 10 dígitos, documentos `CC/CE/NIT/PAS/TI`. Otro país = agregar un registro (regex + catálogo de ciudades), sin cambiar código | Decisión Iker | Confirmada |
| RN-102 | La ciudad se guarda como `{ pais, sistema, codigo }` (ej. `{ "CO", "DANE", "05001000" }`) | Decisión Iker | Confirmada |
| RN-103 | Zona horaria: hoy las fechas se escriben restando 5 horas a UTC (`CR/.../CargaInterUseCase.cs:216-218`). Debe pasar a ser una propiedad del país de la tienda | Recomendación | Pendiente (RN-P-12) |

---

## 10. Pendientes y riesgos

### Pendientes de datos o decisión

| Id | Pendiente | Responsable |
| --- | --- | --- |
| RN-P-01 | **Resuelto 2026-10-24:** mínimo de 45 000 COP para todas las transportadoras, con o sin recaudo (RN-012). Se vuelve a abrir solo si aparecen problemas | Iker |
| RN-P-02 | Límites de peso y dimensiones, y largos máximos de texto de Inter y de las demás transportadoras (RN-015, RN-019) | Iker |
| RN-P-03 | Nombres, tipo y largo de los 2–3 campos de `datosAdicionales` de las tiendas propias (RN-024) | Iker |
| RN-P-04 | Dropshipping: Iker definió "origen de la tienda; si no, el del proveedor del producto" (RN-033). Confirmar que en dropshipping el paquete sale físicamente de la tienda y no del proveedor, porque la sucursal remitente (RN-032) sí sale del proveedor | Iker |
| RN-P-05 | Lista inicial de estados anulables: solo `Creado` (backend de hoy) o los tres de `cupo.md` RC-024 (RN-072) | Iker |
| RN-P-06 | Mapeo de `tipoDocumento` a cada transportadora (Inter hoy solo manda `CC`) (RN-017) | Desarrollo, al integrar cada transportadora |
| RN-P-07 | Valores fijos del servidor: `ASESOR`, `CODIGO OFICINA`, `CODIGO REGIONAL`, `SEGURO`, `TELEFONO REMITENTE` (hoy `3112222222` por defecto en TCC) y `FORMA DE PAGO` (RN-023) | Iker / Desarrollo |
| RN-P-08 | Catálogo de mensajes normalizados de errores de Inter (RN-062) | Desarrollo |
| RN-P-09 | Confirmar el efecto de la anulación en inventario (RN-077) | Iker |
| RN-P-10 | Comportamiento del cupo dentro de un lote (RN-087) | Chat de cupo |
| RN-P-11 | Guías encoladas cuando se bloquea un cliente (RN-097) | Iker |
| RN-P-12 | Zona horaria por país (RN-103) | Desarrollo |
| RN-P-13 | `correo`, `diceContener` y `observaciones` (RN-021, RN-022) | Iker |
| RN-P-14 | Unificar: ¿`referencia` también en `POST /v1/guias`, además de `Idempotency-Key`? (RN-010) | Iker |
| RN-P-15 | Migrar estas especificaciones a la base de conocimiento cuando exista | Iker / Claude (recordar) |

### Riesgos aceptados

| Id | Riesgo | Mitigación |
| --- | --- | --- |
| R-01 | La anulación solo marca internamente (RN-070): la guía sigue viva en la transportadora y podría despacharse y cobrarse como una guía real aunque en LogiGho figure `Anulada` y libere cupo | Decisión consciente de Iker. Vigilar con la conciliación de novedades de la transportadora. La matriz (RN-071) permite activar la anulación externa por transportadora sin cambiar el contrato |

### Discrepancias con otros documentos

| Documento | Dice | Esta especificación |
| --- | --- | --- |
| [endpoints.md](endpoints.md) línea 36 | `POST /v1/lotes` "valida todo antes" | Lote parcial con `rechazoId` (RN-081 a RN-084). Actualizar `endpoints.md` |
| [endpoints.md](endpoints.md) línea 34 | `anulacion` "anula la guía en la transportadora y libera cupo" | Solo marca interna (RN-070). Actualizar `endpoints.md` |
| [ApiPublicaGuias.md](ApiPublicaGuias.md) línea 116 | `reglas-negocio.md` "en construcción" | Este documento |

---

## Referencias

- [ApiPublicaGuias.md](ApiPublicaGuias.md) · [endpoints.md](endpoints.md) · [cupo.md](cupo.md) · [hallazgos-lambdas-actuales.md](hallazgos-lambdas-actuales.md)
- [ADR-003 Idempotencia](adr-003-idempotencia.md) · [ADR-004 Etiquetas](adr-004-etiquetas.md)
- Lambdas actuales: `ApiLambdaCrearGuia`, `ApiLambdaCargarGuias`, `ApiLambdaCrearEtiquetaManual`, `ApiLambdaAnularGuia`
