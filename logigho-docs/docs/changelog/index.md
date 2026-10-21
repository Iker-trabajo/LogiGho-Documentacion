# Hisotial de cambios

Historial de cambios, nuevas funcionalidades y correcciones del sistema LogiGho.

---

## Formato de entrada

```
## [version o fecha] — Descripción corta

### Nuevas funcionalidades
- Descripción de qué se agregó

### Cambios
- Descripción de qué cambió

### Correcciones
- Descripción de bugs corregidos

### Documentación
- Qué se documentó en esta entrega
```

---

## [2026-10-21] — Exportación de datos: límites por formato, varias guías, resumen y celular

### Nuevas funcionalidades

- **Límites por formato** en Pedidos (`PedidosInter`) y Liquidaciones (`LiquidacionesLogigho`):
  CSV 300.000, Excel 130.000, JSON 150.000 y PDF 150.000 registros con máximo 10 columnas.
  Debajo de cada rango de fecha sale cuántos registros trae y solo se pueden elegir los que caben;
  en las fechas personalizadas los días que se pasan salen tachados.
- **Rangos nuevos:** "Este mes" (del día 1 a hoy, por defecto) y "Último mes" (mes anterior completo).
- **Varias guías en un filtro:** los campos numéricos enteros aceptan valores separados por coma o
  pegados de una columna de Excel (máximo 400). Buscando por guía se exporta todo el historial y sin
  límite de registros.
- **Filtro Ecosistema** en Pedidos y Liquidaciones, con **varios ecosistemas** a la vez: se envía
  como la lista de tiendas de esos ecosistemas que el usuario tiene asignadas. Solo salen los
  ecosistemas con tiendas del usuario y se controla que las tiendas quepan en la dirección de la
  exportación.
- **Revisión de guías y preenvíos:** al agregarlos se revisa cuáles existen en las tiendas del
  usuario; si faltan, se puede seguir solo con los que sí existen o corregir. Buscando por preenvío
  también se exporta todo el historial y sin límite.
- **Ayudas al pasar el mouse** (directiva `appAyuda`) en lugar del `title` del navegador; los rangos
  de fecha muestran sus fechas exactas.
- **Teclado en Filtros avanzados:** flechas y Enter para elegir el campo, Enter en el valor agrega el
  filtro y Ctrl+Z deshace el pegado de guías.
- **Permisos por rol** en colecciones (`ListaColecciones.Roles`) y en campos
  (`ConfiguracionCamposColeccion.Roles`).
- **Resumen "Así queda tu exportación"** con el panel "Registros a exportar" (número, barra de
  capacidad verde/naranja/roja, Columnas, Celdas, Aún caben y estado) en todas las colecciones.
- **Buscador de columnas** (sin mayúsculas, tildes ni espacios, con la coincidencia marcada) y
  **lista de formatos** con nivel ALTO / MEDIO / BAJO.
- Accesos **"Ir directo a"** en el encabezado, pastilla **"Ir a exportar"**, aviso **"Primero
  selecciona una colección"** y pregunta al exportar si quedó un **filtro sin agregar**.
- **Mensaje al equipo de desarrollo** cuando se necesita más del límite: llega como notificación de
  la campana con lo que se intentaba exportar, y la respuesta del equipo le llega a quien lo envió.
- **Campana de notificaciones** en el header (reemplaza la campana fija con el "1"): contador de
  nuevas, pestañas Pendientes / Historial (últimas 20), detalle con el contexto, respuesta obligatoria
  al atender una solicitud y "Entendido" para las respuestas. Revisa cada 2 min (rol Desarrollador) o
  5 min (los demás), solo con la pestaña visible. Colección nueva `NotificacionesPlataforma`.
- **Aviso de notificaciones** (`AvisoNotificacionService`): tarjeta arriba a la derecha con icono de
  color, X para cerrar y barra de tiempo.
- **Celular:** listas como paneles desde abajo, calendario del teléfono, columnas en 2 por fila e
  información de uso plegable.
- **Tour guiado** de la pantalla (14 pasos, incluida la campana de notificaciones), con botón que
  late la primera vez.

### Cambios

- Se quitó **"Exportar Servidor"** (botón y flujo en `export-runner`): el backend `exportReportes`
  respondía 504 a los 29 segundos aun con pocos registros.
- Tour guiado (componente compartido): no deja mover la página mientras está abierto, oscurecido de
  una sola pieza y tarjeta centrada en celular. Afecta también a los tours de Pautas y Bodegas.

### Correcciones

- Buscar por guía daba "No hay datos" porque se aplicaba el rango "Este mes"; ahora se busca en todo
  el historial.
- `localStorage` de exportaciones viejas nunca se borraba (no leía bien la hora del `exportId`);
  ahora se borra a los 30 minutos.

### Documentación

- **Export data** actualizada: partes de la pantalla, límites, rangos, panel de registros, varias
  guías, Ecosistema, roles, mensaje al equipo, celular, tour y pruebas.
- **Export runner** sin el flujo de "Exportar Servidor".
- **Tour guiado:** nueva fila en el historial de cambios.
- Nuevas páginas: **Core → Servicio de Notificaciones**, **components → Campana de Notificaciones**
  y **Gestion de datos → Directiva ayuda flotante**.

---

## [2026-10-17] — Propuesta: Sistema de Experiencias y seguridad de accesos

### Documentación
- **Sistema de Experiencias (propuesta, pendiente de aprobación):** qué es, ADR-001, estructura del proyecto,
  crear una vista, crear una experiencia, reglas de cambio entre experiencias, tokens y estilos, seguridad por
  diseño, buenas prácticas, migración de vistas, ejemplo con `tienda/pedidos` y plan de implementación.
- **Seguridad y accesos (propuesta):** modelo objetivo de autorización en el servidor.
- **Plantilla nueva:** documentación de vista del Sistema de Experiencias.
- **Estilos y SCSS:** ahora enlaza al estándar de tokens.

---

## [2026-10-14] — Pauta: validaciones de la vista y nueva importación con previsualización

### Nuevas funcionalidades

- **Previsualización editable al importar:** antes de guardar se ven todas las filas del Excel;
  cada celda se valida mientras se escribe y los errores salen en rojo con un mensaje corto
  ("Solo números", "Obligatorio"). Solo se deja importar cuando no queda ningún error.
- **Corregir automáticamente:** deja solo los dígitos en los montos e Idtienda
  (`1.500.000` → `1500000`); las celdas corregidas quedan en verde con "Antes: …".
- **Verificación de tiendas:** el nombre de la tienda se llena solo con el Idtienda (una sola
  consulta para todo el archivo) y se marca "Tienda no existe", "Tienda inactiva" o
  "No es tu tienda".
- Filtro **Solo errores**, **Eliminar fila** con Deshacer, **Agregar fila** y confirmación
  al cerrar con cambios sin importar.
- Zona para **arrastrar y soltar** el Excel y errores del archivo dentro del modal con temblor.
- **Tour guiado** en la vista y en el modal.
- **Celular:** la tabla de Pauta se ve como tarjetas con el total de cada registro, el
  calendario y el modal salen como hoja desde abajo.

### Cambios

- La vista carga **todas** las páginas de pauta (8 en paralelo, Zstd y solo los campos de la
  tabla), con barra "Cargando registros… X de N" y aviso con Reintentar si alguna no llega.
  El contador, los totales, el filtro y la descarga usan todos los registros.
- Los montos solo aceptan números enteros, al editar y al importar (nueva directiva
  `appCampoSoloNumeros` con aviso toast).
- Encabezados exactos de la plantilla: se rechazan columnas faltantes, no permitidas,
  repetidas o con datos sin nombre.
- Organización del módulo: llamadas al backend en `PautaRepository`, lectura y validación del
  Excel en servicios, modelos en `models/` y estilos partidos en `estilos/`.

### Correcciones

- El campo `Otros` se leía como `Otro`: la columna, el total y la descarga salían vacíos.
- Los totales ya no se vuelven `NaN` con montos viejos mal guardados (ej. `1.500.000`); esos
  valores se ven en rojo para corregirlos.
- Al guardar una fila con un monto inválido ya no queda `NaN` en la tabla.

### Documentación

- **Vista Pauta** y **Importacion Pauta** actualizadas.
- Nuevas páginas en **Frontend → Vistas → Tienda → Pauta:** Modelos, Repositorio, Lectura de
  Excel, Validacion de filas, Aviso toast y Directiva solo numeros.

---

## [2026-09-21] — Dominio Datos: API backend para publicar/versionar reportes analíticos

### Nuevas funcionalidades

- **Nueva Lambda `ApiLambdaPublicarReporteAnalytics`** (dominio Datos): mueve al backend
  el flujo de publicar/versionar reportes analíticos, que hasta ahora corría 100% en el
  cliente (Angular subía directo a S3/Mongo, sin control server-side). Presigned URL de
  S3 en 2 pasos (`solicitar-carga` → `PUT` directo a S3 → `confirmar-carga`) para evitar
  el límite de payload de API Gateway con reportes de hasta 15 MB. Port literal de las 8
  reglas de validación de contenido del sanitizador Angular (`endurecerHtml`), 8/8
  verificadas idénticas contra el archivo fuente.
- Nuevo dominio **Datos** en PreProd: CloudFormation (`datos-infra-preprod.yaml`) +
  pipeline GitHub Actions (`deploy-datos-preprod.yml`) desde cero, siguiendo la receta
  estándar del repo. Primera lambda del repo que crea su propio rol IAM por CFN (los
  demás dominios reusan roles pre-creados a mano).
- Nuevo campo `Autor` en `CargasReportesAnalytics`, para auditoría legible (email/username)
  sin tener que cruzar el `sub` contra `Users`.

### Cambios

- **Modelo de autenticación rediseñado** durante la construcción: de un diseño con
  Cognito Authorizer + colección `IntegracionesExternas` propia, a decodificar el `sub`
  del JWT sin verificar firma y exigir rol (`Jefe Datos`/`Desarrollador`/`CEO`) en
  `Users` — consistente con cómo funciona el resto de la plataforma hoy (ningún endpoint
  tiene Authorizer nativo). Ver ADR-001.
- `MONGODB_CONNECTION_STRING` migrado a cifrado AES256-ECB (`Seguridad.Encripcion.EncripcionAES`),
  estándar ya usado en el resto del repo para secretos en variables de entorno.
- Respuestas de error migradas de objetos anónimos a records tipados (`ErrorResponse`,
  `ValidacionRechazadaResponse`), consistente con el resto de lambdas del repo.

### Correcciones

- **Bug crítico de despliegue**: el handler usaba tipos de evento API Gateway V1
  (`APIGatewayProxyRequest`) contra una integración real V2 (`PayloadFormatVersion:
  "2.0"`) — `request.HttpMethod` quedaba siempre vacío y toda petición real caía en
  `405 RUTA_NO_ADMITIDA`, aunque los 96 tests unitarios (con eventos V1 simulados)
  pasaban en verde. Detectado en la primera prueba real contra PreProd con Postman.
  Migrado a `APIGatewayHttpApiV2ProxyRequest`/`Response`.
- `ApiLambdaGetObjectMetadataAOT` (necesaria para que Angular lea el HTML ya publicado)
  no existía en la cuenta AWS de PreProd — replicada manualmente desde Producción
  (mismo paquete `dotnet8`) con ruta `POST /getObject` en el Gateway de PreProd.
  Pendiente formalizar en IaC.

### Documentación

- Nueva sección **Backend → Lambdas .NET → LogiGho → ApiLambdaPublicarReporteAnalytics**:
  visión general con contrato completo de los 2 endpoints, flujo punta a punta, las 8
  reglas de validación, casos de uso (Solicitar Carga, Confirmar Carga), modelos de
  dominio, repositorios, servicios de infraestructura, y ADR-001 (autenticación).

---

## [2026-09-16] — Liquidaciones: motor Dinámico (TCC) conviviendo con el motor Legacy

### Nuevas funcionalidades

- **Motor Dinámico de liquidaciones**, dentro de la misma Lambda `ApiLambdaLiquidacionesLogighoAOT` (patrón Strangler Fig, ver ADR-001): dominio limpio (modelos, calculadoras, políticas de peso, validadores), configurable por transportadora vía `ConfiguracionLiquidacionTransportadora` (`Dinamica`/`Legacy`/`Sombra`/`Deshabilitado`). Liquida hoy TCC; el motor Legacy sigue intacto para Interrapidísimo, Envía y Servientrega.
- **Módulo frontend "Costos Transportadora"** (`director-de-operaciones`): Hub de transportadoras + panel de 5 pestañas (Configuración General, Trayectos, Tarifas por Tienda, Simulador de Liquidación, Auditoría de Excepciones) para configurar el motor Dinámico sin desplegar código.
- **Dominio AWS `financiero`** en PreProd: Lambda + Step Function `Financiero-PreProd`, para ejecutar liquidaciones de prueba (todas las pendientes, o una lista puntual de guías) sin tocar producción.

### Correcciones

- `TarifaTransportadoraTienda.Trayectos` e `IncrementosPeso` cambiados de `IReadOnlyList<T>` a `List<T>` — Newtonsoft no puede poblar una `ReadOnlyCollection` al deserializar directo desde Mongo (`JsonSerializationException` real visto en CloudWatch). Se agregaron 3 tests de regresión que deserializan JSON con forma real, cerrando el hueco de cobertura que dejó pasar el bug.
- `URL_SERVICIO_AWS` agregada al template de `financiero-infra-preprod.yaml` — faltaba para que el camino legacy no-TCC (`Generico.consumoGenerico`) pudiera actualizar inventario cuando se mezclan pedidos legacy y TCC en la misma ejecución.

### Documentación

- Nueva sección **Backend → Lambdas .NET → LogiGho → ApiLambdaLiquidacionesLogighoAOT**: visión general con diagrama de flujo estilo BPMN, motor Legacy, motor Dinámico completo (orquestador, modelos, calculadoras, políticas de peso, validación, infraestructura), despliegue y operación en PreProd, ADR-001.
- Nueva sección **Frontend → Director de Operaciones → Costos Transportadora**: componente principal, las 5 pestañas, modelos, servicio de estado, ADR-001.
- Nueva página **Frontend → Analytics → Liquidaciones (Power BI)**.
## [2026-09-09] — Cotizador con/sin recaudo: Envía real + Strategy + reglas de forma de pago

### Nuevas funcionalidades

- **Envía ahora cotiza real** en `ApiLambdaCotizarEnvia` — antes el orquestador estimaba geográfico porque el request no traía los campos de cuenta requeridos y no se distinguía el error de negocio del éxito.
- **Con/sin recaudo en las 3 transportadoras** (`Common.ConRecaudo`): antes solo existía visualmente para Envía y ni siquiera se aplicaba de verdad.
- Backend: patrón **Strategy** (`ICotizadorProveedor` + `RegistroCotizadores`) para desacoplar cada transportadora del orquestador. `DirectFunction` bajó de ~200 a ~55 líneas.
- Backend: `EnviaCuentaResolver` resuelve la cuenta de Envía (2 cuentas globales) según `ConRecaudo` — con la **semántica invertida** de Envía (Crédito = con recaudo, Contraentrega = sin recaudo).
- Front: paso 3 del modal de creación de pedidos extraído a `PasoCotizacionComponent`, con badge de tarifa más económica y botón de desglose crudo (gateado por rol, no por backend).
- Front: nuevo módulo `reglas-modalidad-pago.ts` — single source of truth de cómo el switch de recaudo se traduce a "Forma de pago" por transportadora en el paso 4 (Guía).

### Correcciones

- Envía respondía `200 OK` con error de negocio (peso fuera de rango, etc.) y el orquestador lo leía como cotización válida con flete $0 — nueva `EnviaLiquidacionException`.
- Interrapidísimo: el cliente HTTP mandaba `ValorContraPago` en el campo que debía llevar `ValorDeclarado` — separados en el DTO y el mapeo. Eliminado `IdFormaPago` (campo muerto).
- **Bug de producción**: guía de Envía quedaba forzada a `FORMA DE PAGO = CREDITO` / `APLICA CONTRA PAGO = NO` sin importar el switch del usuario, por un bloque de cálculo duplicado en `generarGuia()`. Corregido centralizando la regla en `reglas-modalidad-pago.ts`.

### Documentación

- Actualizados `ApiLambdaOrquestadorCotizaciones.md`, `ApiLambdaCotizarEnvia.md`, `ApiLambdaCotizarInterrapidisimo.md`, `ApiLambdaGenerarCotizacion.md`.
- Nuevo [ADR-001 — Strategy para cotizadores y semántica de recaudo](../backend/lambdas-dotnet/aplicacion/Cotizacion/FuncionesCotizar/ApiLambdaOrquestadorCotizaciones/ADR-001-strategy-cotizadores.md).
- Nuevas páginas front: [paso-cotizacion.md](../frontend/components/paso-cotizacion.md), [cotizacion-service.md](../frontend/core/cotizacion-service.md), [reglas-modalidad-pago.md](../frontend/core/reglas-modalidad-pago.md). Actualizado `modal-creacion-pedidos.component.md`.

---

## [2026-09-02] — DevolucionesMasivo: confiabilidad de Inter (token real, reintentos, fallback cruzado)

### Correcciones

- **Token de Inter vencía en 20 minutos según el código, en 30 segundos según Inter real** (confirmado decodificando el JWT que devuelve `GenerarTokenTemporal`) — causaba 401 masivos en cargas de más de 30s de duración. `_tokenVence` bajado a 20s.
- Fallos HTTP de `ClienteInter` (rastreo y estados) no dejaban ningún rastro en logs cuando Inter respondía distinto de 2xx — ahora se loguea `statusCode` y cuerpo de la respuesta en cada intento fallido.
- Sin reintentos ante `429`/`5xx` de Inter: ahora hasta 4 intentos con backoff exponencial + jitter, respetando `Retry-After`.
- Tráfico sostenido sin pausas entre tandas (`SemaphoreSlim` sin cortes) podía seguir gatillando 429 con lotes de 200+ guías — ahora hay pausa configurable entre tanda y tanda (`PAUSA_ENTRE_PETICIONES_INTER_MS`).
- Guías rechazadas por la transportadora asignada (`ErrorConsultaExterna`, ej. 400 explícito) o sin ningún formato reconocido (`GuiaNoExisteEnSistema`) no se reintentaban contra otra transportadora — solo cubría `SinGuiaOriginalEnRespuesta`. Ahora los 3 motivos disparan el fallback cruzado, con Inter primero por volumen.

### Cambios

- `CONCURRENCIA_INTER` por defecto bajada de 10 a 5 en código y en ambos entornos (prod/preprod).

### Documentación

- [ADR-005](../backend/lambdas-dotnet/aplicacion/Logigho/ApiLambdaDevolucionesMasivo/adr/ADR-005-confiabilidad-inter.md) con el diagnóstico completo. Actualizados `clientes-transportadora.md`, `worker-handler.md` y la tabla de variables de entorno del módulo.

---

## [2026-07-27] — Pancake: doble escritura de páginas, fix de ventana y endpoint on-demand

### Nuevas funcionalidades

- **Endpoint on-demand** `ApiLambdaConsultarEstadisticasPagina` (API Gateway `POST`): trae estadísticas frescas de una página (hoy / ayer / rango personalizado) **sin persistir**, con fallback de token de página.
- `ApiLambdaListarPaginasPancake` ahora hace **doble escritura** Mongo + Aurora MySQL.

### Correcciones

- Ventana `cierre_dia_anterior`: `until` corregido de `00:00:00` del día siguiente a `**23:59:59`** del mismo día — Pancake ya no arrastra gasto del otro día.

### Documentación

- Nueva página **Endpoint on-demand (API)** en Backend → Integración Pancake; actualizadas las páginas de `ListarPaginasPancake` (SQL) y `CalcularVentanaTiempo` (fix `until`).

---

## [2026-07-13] — Integración Pancake (estadísticas de campañas)

### Nuevas funcionalidades

- Pipeline serverless de **5 lambdas .NET 8** que recolecta las estadísticas de campañas de Pancake (`pages.fm`) 4 veces al día (7am/9am/2pm/5pm hora Colombia), orquestado con **AWS Step Functions** y agendado con **EventBridge Scheduler**.
- **Doble escritura** de estadísticas por campaña: MongoDB + **RDS Aurora MySQL** (patrón Composite, best-effort, UPSERT idempotente, modo inerte plug-and-play).

### Documentación

- Nueva sección **Backend → Integración Pancake**: visión general con diagrama de arquitectura, una página por lambda, orquestación (Step Functions + ASL + EventBridge) y guía de operación (ejecución manual, agregar franjas, monitoreo y troubleshooting).

