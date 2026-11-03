# Rutas de la plataforma: inventario y clasificación (spec 009, tarea 1)

**Fuente:** API Gateway `7elgyten88`, stage `Produccion`, cuenta 522814697657, leído solo en lectura el 2026-10-31 (`get-resources --embed methods` y `get-integration` por método).
**Estado de la clasificación:** *propuesta* hecha por patrón de nombre; cada fila se confirma leyendo su handler y los logs de consumidores antes de cerrar la ruta (spec 009, tarea 2).

## Resumen

- **134 métodos** (sin contar `OPTIONS`) en 87 rutas.
- **104 métodos sin authorizer** en el gateway; **30 con authorizer** de Cognito sobre el header `Token`.
- 0 rutas quedaron sin clasificar y se revisan a mano.

!!! warning "Sin authorizer no significa abierto, pero hoy casi todo depende de la lambda"
    Un método sin authorizer solo está protegido si su lambda valida identidad. Dinero (`/wallet`, `/liquidaciones`, `/retirosDatos`), permisos (`/roles`, `/permissions`, `/module`) y datos (`/tiendas`, `/pedidos`) figuran **sin authorizer**. Cerrarlas es la tanda 1 a 3 de la spec 009 y se hace de a una, con modo de solo registro, observación y reversión lista (política M-195).

## Tandas (de mayor a menor riesgo)

1. Dinero. 2. Permisos. 3. Datos y archivos. 4. Procesos. 5. `/metodoGenerico`. Las rutas del dominio autenticación ya validan identidad dentro de la lambda (spec 007).

## Inventario

| Ruta | Métodos | Lambda | Gateway hoy | Clase propuesta | Tanda | Nota |
|---|---|---|---|---|---|---|
| `/actualizaInventarioLotes` | POST | ApiLambdaInventarioPorCarga | sin authorizer | Usuario (procesos) | 4 | Procesos operativos y de integración |
| `/actualizacionRetiro` | POST | ApiLambdaRetirosHokoAOT | sin authorizer | Webhook de tercero (Hoko) a confirmar | 1 | Mueve dinero: validar firma o credencial de servicio |
| `/actualizarInventario` | POST | ApiLambdaInventarioPorGuia | authorizer | Usuario (procesos) | 4 | Procesos operativos y de integración |
| `/authAzure` | POST | ApiLambdaTokenAzureAOT | sin authorizer | Servicio (token de integración) | 3 | Entrega tokens de terceros; confirmar quién los llama |
| `/authHoko` | POST | ApiLambdaTokenHokoAOT | sin authorizer | Servicio (token de integración) | 3 | Entrega tokens de terceros; confirmar quién los llama |
| `/cambio-pass` | POST | ApiLambdaCambioPasswordCognito | sin authorizer | Usuario (identidad validada en la lambda) | hecho; **obsoleta** (la reemplaza la recuperación) | Dominio autenticacion; Token JWT + rol desde Users; spec 007 |
| `/cambio-pass/codigo` | POST | ApiLambdaCambioPasswordCognito | sin authorizer | Usuario (identidad validada en la lambda) | hecho en PreProd | Envía el código al correo de la cuenta; spec 008 y M-214 |
| `/autenticacion/registro/solicitudes` | POST | ApiLambdaRegistro | sin authorizer | Pública por diseño | hecho en PreProd | Responde 202 igual exista o no el correo; límite por correo; spec 008 |
| `/autenticacion/registro/solicitudes/reenvio` | POST | ApiLambdaRegistro | sin authorizer | Pública por diseño | hecho en PreProd | 60 s entre envíos, 3 por hora; spec 008 |
| `/autenticacion/registro/confirmaciones` | POST | ApiLambdaRegistro | sin authorizer | Pública por diseño | hecho en PreProd | Código de un solo uso, 5 intentos; crea la cuenta con rol `Red social`; spec 008 |
| `/autenticacion/recuperacion/solicitudes` | POST | ApiLambdaRegistro | sin authorizer | Pública por diseño | hecho en PreProd (PR 579) | Responde 202 igual exista o no la cuenta; manda el código al correo; 60 s entre envíos y 3 por hora; olvidé mi contraseña y cambio desde el perfil |
| `/autenticacion/recuperacion/confirmaciones` | POST | ApiLambdaRegistro | sin authorizer | Pública por diseño | hecho en PreProd (PR 579) | Código de un solo uso, 5 intentos; pone la contraseña nueva y cierra las sesiones |
| `/autenticacion/roles-asignables` | GET | ApiLambdaUser | sin authorizer | Usuario (identidad validada en la lambda) | hecho en PreProd | Roles que quien consulta puede asignar |
| `/autenticacion/usuarios/{usuarioId}/acceso` | POST | ApiLambdaUser | sin authorizer | Usuario (identidad validada en la lambda) | hecho en PreProd | Comercial asigna acceso a cualquier cliente; se quita `Red social`; con traza |
| `/cargaDevoluciones` | POST | ApiLambdaGestionDevoluciones | sin authorizer | Usuario (procesos) | 4 | Procesos operativos y de integración |
| `/cargaHoko` | GET, POST, PUT | ApiLambdaCrudCargaHokoAOT | sin authorizer | Usuario (dinero) | 1 | Mayor riesgo: lectura y escritura de saldos y retiros |
| `/cargaPedidos` | ANY | ApiLambdaCrudCargaPedidosAOT | sin authorizer | Usuario (procesos) | 4 | Procesos operativos y de integración |
| `/cargaPreenvios` | POST | ApiLambdaCargarGuias | sin authorizer | Usuario (procesos) | 4 | Procesos operativos y de integración |
| `/compraEstado` | GET | ApiLambdaCalculosPedidosAOT | sin authorizer | Usuario (datos) | 3 | Datos de tiendas y pedidos; aislamiento por tienda |
| `/consultaEstadosServientrega` | POST | ApiLambdaEstadoGuias | authorizer | Usuario (procesos) | 4 | Procesos operativos y de integración |
| `/countColeccion` | POST | LambdaCountColeccion | sin authorizer | Usuario (utilidad de datos) | 3 | Cuenta documentos o escribe auditoría: exigir identidad |
| `/creacionGuia` | POST | ApiLambdaCrearGuia | sin authorizer | Usuario (procesos) | 4 | Procesos operativos y de integración |
| `/crearGuiaServientrega` | POST | ApiLambdaCrearEnvio | authorizer | Usuario (procesos) | 4 | Procesos operativos y de integración |
| `/createUser` | POST | ApiLambdaCreateUserCognitoAOT | sin authorizer | Usuario (identidad validada en la lambda) | hecho | Dominio autenticacion; Token JWT + rol desde Users; spec 007 |
| `/cuentasbancarias` | GET, POST, PUT | ApiLambdaCuentasBancarias | sin authorizer | Usuario (dinero) | 1 | Mayor riesgo: lectura y escritura de saldos y retiros |
| `/devolucionesMasivo/estado` | GET | ApiLambdaDevolucionesMasivoEstado | authorizer | Usuario (procesos) | 4 | Procesos operativos y de integración |
| `/devolucionesMasivo/iniciar` | POST | ApiLambdaDevolucionesMasivoIniciar | authorizer | Usuario (procesos) | 4 | Procesos operativos y de integración |
| `/enviosms` | POST | ApiLambdaNotificacionesSnsAOT | sin authorizer | Usuario (procesos) | 4 | Procesos operativos y de integración |
| `/exportArchivo` | POST | LambdaExportTipoReportes | authorizer | Usuario (archivos S3) | 3 | Acceso a objetos y reportes |
| `/exportData` | GET | ApiLambdaExportLogighoAOT | authorizer | Usuario (archivos S3) | 3 | Acceso a objetos y reportes |
| `/exportReportes` | POST | LambdaExportReportes | authorizer | Usuario (archivos S3) | 3 | Acceso a objetos y reportes |
| `/generarTokenServientrega` | POST | ApiLambdaGenerarTokenServientrega | sin authorizer | Servicio (token de integración) | 3 | Entrega tokens de terceros; confirmar quién los llama |
| `/getMultiObject` | POST | ApiLambdaGetMultiObjectZipAOT | sin authorizer | Usuario (archivos S3) | 3 | Acceso a objetos y reportes |
| `/getObject` | POST | ApiLambdaGetObjectMetadataAOT | sin authorizer | Usuario (archivos S3) | 3 | Acceso a objetos y reportes |
| `/guiasHoko` | DELETE, PATCH, POST | ApiLambdaGuiasHoko | authorizer / sin authorizer | Usuario (dinero) | 1 | Mayor riesgo: lectura y escritura de saldos y retiros |
| `/insertarAuditoria` | ANY | ApiLambdaInsertarAuditoria | sin authorizer | Usuario (utilidad de datos) | 3 | Cuenta documentos o escribe auditoría: exigir identidad |
| `/inventario/despachos` | GET | ApiLambdaInventarioDespachos | sin authorizer | Ya validada en lambda (a confirmar) | confirmar | Según specs de inventario y métricas; confirmar y documentar |
| `/inventario/suscripciones` | GET, POST | ApiLambdaInventarioSuscripciones | sin authorizer | Ya validada en lambda (a confirmar) | confirmar | Según specs de inventario y métricas; confirmar y documentar |
| `/inventario/suscripciones/prueba` | POST | ApiLambdaInventarioSuscripciones | sin authorizer | Ya validada en lambda (a confirmar) | confirmar | Según specs de inventario y métricas; confirmar y documentar |
| `/inventario/suscripciones/{id}` | DELETE, GET, PUT | ApiLambdaInventarioSuscripciones | sin authorizer | Ya validada en lambda (a confirmar) | confirmar | Según specs de inventario y métricas; confirmar y documentar |
| `/inventario/suscripciones/{id}/prueba` | POST | ApiLambdaInventarioSuscripciones | sin authorizer | Ya validada en lambda (a confirmar) | confirmar | Según specs de inventario y métricas; confirmar y documentar |
| `/inventario/trazabilidad` | GET | ApiLambdaInventarioTrazabilidad | sin authorizer | Ya validada en lambda (a confirmar) | confirmar | Según specs de inventario y métricas; confirmar y documentar |
| `/liquidacionInversor` | POST | ApiLambdaLiquidacionesInversorAOT | authorizer | Usuario (dinero) | 1 | Mayor riesgo: lectura y escritura de saldos y retiros |
| `/liquidacionManual` | POST | ApiLambdaLiquidacionesLogighoAOT | authorizer | Usuario (dinero) | 1 | Mayor riesgo: lectura y escritura de saldos y retiros |
| `/liquidaciones` | DELETE, GET, PATCH, POST, PUT | ApiLambdaCrudLiquidacionesAOT | sin authorizer | Usuario (dinero) | 1 | Mayor riesgo: lectura y escritura de saldos y retiros |
| `/liquidacionesHoko` | POST | ApiLambdaLiquidacionesHoko | sin authorizer | Usuario (dinero) | 1 | Mayor riesgo: lectura y escritura de saldos y retiros |
| `/listObjects` | POST | ApiLambdaGetListObjectsAOT | sin authorizer | Usuario (archivos S3) | 3 | Acceso a objetos y reportes |
| `/metodoGenerico` | ANY | ApiLambdaCrudGenericoAOT | authorizer | Usuario (genérica) | 5 | Con authorizer; falta lista blanca por rol y sin borrado de colección |
| `/metricas/entrega-ciudad` | GET | ApiLambdaMetricasEntregaCiudad | sin authorizer | Ya validada en lambda (a confirmar) | confirmar | Según specs de inventario y métricas; confirmar y documentar |
| `/metricas/entrega-ciudad/configuracion` | GET, PUT | ApiLambdaMetricasEntregaCiudad | sin authorizer | Ya validada en lambda (a confirmar) | confirmar | Según specs de inventario y métricas; confirmar y documentar |
| `/metricas/entrega-ciudad/corridas` | GET, POST | ApiLambdaMetricasEntregaCiudad | sin authorizer | Ya validada en lambda (a confirmar) | confirmar | Según specs de inventario y métricas; confirmar y documentar |
| `/metricas/entrega-ciudad/corridas/{id}` | GET | ApiLambdaMetricasEntregaCiudad | sin authorizer | Ya validada en lambda (a confirmar) | confirmar | Según specs de inventario y métricas; confirmar y documentar |
| `/module` | GET, POST, PUT | ApiLambdaModule | sin authorizer | Usuario (permisos) | 2 | Escalada de privilegios si se escribe sin control |
| `/notificacionRetiro` | GET, POST | ApiLambdaCrudRetirosAOT, ApiLambdaInitNotRetirosHokoAOT | sin authorizer | Webhook de tercero (Hoko) a confirmar | 1 | Mueve dinero: validar firma o credencial de servicio |
| `/obtenerDatosPaginasPancake` | POST | ApiLambdaConsultarEstadisticasPagina | authorizer | Usuario (procesos) | 4 | Procesos operativos y de integración |
| `/obtenerEtiquetaServientrega` | POST | ApiLambdaImprimirGuia | authorizer | Usuario (procesos) | 4 | Procesos operativos y de integración |
| `/obtenerTokenAuth` | POST | ApiLambdaTokenCognitoAOT | sin authorizer | Servicio (credenciales de máquina) | M-194 | Consumen servicios, no el front; endurecida sin cambiar contrato |
| `/obtenerTrazaInter` | POST | ApiLambdaSigueTuEnvio | authorizer | Usuario (procesos) | 4 | Procesos operativos y de integración |
| `/orquestadorCotizacion` | POST | ApiLambdaOrquestadorCotizaciones | sin authorizer | Servicio interno a confirmar | 4 | Orquesta cotizaciones |
| `/parametros` | GET, POST, PUT | ApiLambdaCrudParametrosAOT | sin authorizer | Usuario (procesos) | 4 | Procesos operativos y de integración |
| `/pedidos` | GET, POST, PUT | ApiLambdaCrudPedidosAOT | sin authorizer | Usuario (datos) | 3 | Datos de tiendas y pedidos; aislamiento por tienda |
| `/pedidosInter` | GET | ApiLambdaCrudPedidosInterAOT | authorizer | Usuario (datos) | 3 | Datos de tiendas y pedidos; aislamiento por tienda |
| `/permissions` | GET, POST, PUT | ApiLambdaPermission | sin authorizer | Usuario (permisos) | 2 | Escalada de privilegios si se escribe sin control |
| `/productos` | ANY | ApiLambdaCrudProductosAOT | sin authorizer | Usuario (datos) | 3 | Datos de tiendas y pedidos; aislamiento por tienda |
| `/putObject` | PUT | ApiLambdaPutObjectAOT | sin authorizer | Usuario (archivos S3) | 3 | Acceso a objetos y reportes |
| `/registerContact` | ANY | ApiLambdaCrudRegistroClienteAOT | sin authorizer | Pública por diseño | hecho | Solo POST validado; spec 007 |
| `/reportes-analytics/confirmar-carga/{cargaId}` | POST | ApiLambdaPublicarReporteAnalytics | authorizer | Usuario (procesos) | 4 | Procesos operativos y de integración |
| `/reportes-analytics/solicitar-carga` | POST | ApiLambdaPublicarReporteAnalytics | authorizer | Usuario (procesos) | 4 | Procesos operativos y de integración |
| `/resumenSaldoHoko` | ANY | ApiLambdaCrudResumenSaldoHokoAOT | sin authorizer | Usuario (dinero) | 1 | Mayor riesgo: lectura y escritura de saldos y retiros |
| `/resumenSaldoHokoInit` | POST | ApiLambdaResumenSaldoHoko | sin authorizer | Servicio interno a confirmar | 1 | Nombre 'Init': probablemente la invoca otro proceso, no el front |
| `/retirosDatos` | ANY | ApiLambdaCrudRetirosAOT | sin authorizer | Usuario (dinero) | 1 | Mayor riesgo: lectura y escritura de saldos y retiros |
| `/retirosTienda` | ANY | ApiLambdaCrudRetirosTiendaHokoAOT | sin authorizer | Usuario (dinero) | 1 | Mayor riesgo: lectura y escritura de saldos y retiros |
| `/retirosTiendaInit` | POST | ApiLambdaRetirosTiendaHoko | sin authorizer | Servicio interno a confirmar | 1 | Nombre 'Init': probablemente la invoca otro proceso, no el front |
| `/roles` | GET, POST, PUT | ApiLambdaRole | sin authorizer | Usuario (permisos) | 2 | Escalada de privilegios si se escribe sin control |
| `/saldoDispHokoInit` | POST | ApiLambdaSaldoDisponibleHoko | sin authorizer | Servicio interno a confirmar | 1 | Nombre 'Init': probablemente la invoca otro proceso, no el front |
| `/saldoDisponibleHoko` | DELETE, GET, PATCH, POST, PUT | ApiLambdaCrudSaldoDisponibleHokoAOT | sin authorizer | Usuario (dinero) | 1 | Mayor riesgo: lectura y escritura de saldos y retiros |
| `/sincronizar` | POST | ApiLambdaSincronizacionSqlMongo | authorizer | Usuario (procesos) | 4 | Procesos operativos y de integración |
| `/tarjetas` | ANY | ApiLambdaCrudTarjetasColoca | authorizer | Usuario (dinero, tarjetas) | 1 | Ya con authorizer; falta permiso por rol y tienda |
| `/tarjetas/datosTarjeta` | POST | ApiLambdaGenerarUrlTarjetaColoca | authorizer | Usuario (dinero, tarjetas) | 1 | Ya con authorizer; falta permiso por rol y tienda |
| `/tarjetas/recarga` | POST | ApiLambdaCardTopupWithdrawal | authorizer | Usuario (dinero, tarjetas) | 1 | Ya con authorizer; falta permiso por rol y tienda |
| `/tarjetas/saldos` | POST | ApiLambdaUpsertCardBalance | authorizer | Usuario (dinero, tarjetas) | 1 | Ya con authorizer; falta permiso por rol y tienda |
| `/tarjetas/token` | POST | ApiLambdaGenerarTokenIdentidadColoca | authorizer | Usuario (dinero, tarjetas) | 1 | Ya con authorizer; falta permiso por rol y tienda |
| `/tarjetas/transacciones` | POST | ApiLambdaUpsertCardTransactions | authorizer | Usuario (dinero, tarjetas) | 1 | Ya con authorizer; falta permiso por rol y tienda |
| `/tarjetas/walletBalance` | GET | ApiLambdaObtenerBalanceWalletColoca | authorizer | Usuario (dinero, tarjetas) | 1 | Ya con authorizer; falta permiso por rol y tienda |
| `/tiendadocuments` | GET, POST, PUT | ApiLambdaTiendaDocuments | sin authorizer | Usuario (datos) | 3 | Datos de tiendas y pedidos; aislamiento por tienda |
| `/tiendas` | DELETE, GET, POST, PUT | ApiLambdaCrudTiendaAOT | sin authorizer | Usuario (datos) | 3 | Datos de tiendas y pedidos; aislamiento por tienda |
| `/transaccionesWallet` | GET, POST, PUT | ApiLambdaTransWalletAOT | authorizer / sin authorizer | Usuario (dinero) | 1 | Mayor riesgo: lectura y escritura de saldos y retiros |
| `/transportadora` | GET, POST, PUT | ApiLambdaCrudTransportadorasAOT | sin authorizer | Usuario (procesos) | 4 | Procesos operativos y de integración |
| `/users` | GET, POST, PUT | ApiLambdaUser | sin authorizer | Usuario (identidad validada en la lambda) | hecho | Dominio autenticacion; Token JWT + rol desde Users; spec 007 |
| `/validacionPedidos` | POST | ApiLambdaValidacionesCarga | sin authorizer | Usuario (procesos) | 4 | Procesos operativos y de integración |
| `/wallet` | GET, POST, PUT | ApiLambdaCrudWalletAOT | sin authorizer | Usuario (dinero) | 1 | Mayor riesgo: lectura y escritura de saldos y retiros |
| `/walletHoko` | DELETE, GET, PATCH, POST, PUT | ApiLambdaCrudWalletHokoAOT | authorizer / sin authorizer | Usuario (dinero) | 1 | Mayor riesgo: lectura y escritura de saldos y retiros |
| `/walletHokoInit` | POST | ApiLambdaWalletHoko | sin authorizer | Servicio interno a confirmar | 1 | Nombre 'Init': probablemente la invoca otro proceso, no el front |
| `/webHookShopify` | POST | ApiLambdaRecibeWebHook | sin authorizer | Webhook de tercero | 4 | Debe validar la firma del proveedor |
| `/webhooks/ws-inter/flujo-guia` | POST | ApiLambdaNovedadesInter | sin authorizer | Webhook de tercero | 4 | Debe validar la firma del proveedor |
