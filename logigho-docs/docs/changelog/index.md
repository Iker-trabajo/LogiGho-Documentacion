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

## [2026-10-09] — Layout: rediseño del menú lateral y la barra superior

### Nuevas funcionalidades

- **Buscador del menú** con atajo **Ctrl + B** (**Cmd + B** en Mac): abre la barra con el cursor
  en el buscador, los resultados salen agrupados por sección (ej. "Mi Tienda › Pedidos Masivos")
  y Escape borra y cierra.
- **Celular:** el menú es un cajón completo (300 px u 85 % de la pantalla) que se cierra al
  elegir una opción o deslizando el dedo a la izquierda. La barra superior muestra ☰, logo y
  acciones sin descuadrarse.

### Cambios

- El menú se abre con la clase `abierta` en vez de `:hover`: solo si el mouse se queda encima
  150 ms, con 120 ms de margen antes de cerrarse. Ya no parpadea.
- Apertura por capas (logo con máscara, buscador y textos en cascada, flechas al final), barrita
  azul en la sección actual, opción activa en morado y subgrupos con línea de árbol.
- Barra superior blanca de 56 px, botones redondos lila y píldora de usuario con avatar.
- El comportamiento del menú pasó a la nueva directiva `SidebarInteraccionDirective` y los estilos
  del layout a la carpeta `estilos/` (7 archivos por tema).
- **Seguridad:** `AuthGuard` solo deja pasar sin sesión la ruta exacta `/export-runner`; antes
  bastaba con que apareciera en cualquier parte de la URL, incluido el hash.

### Correcciones

- Clics perdidos: si el menú se mueve mientras se hace clic, la opción igual se abre.
- "Primer clic gastado" al abrir un grupo después de cerrar la barra.
- En celular había que tocar dos veces para ver el menú completo.

### Documentación

- Nueva sección **Frontend → Layout** con **Menú lateral (Default Layout)** y **Barra superior
  (Default Header)**.
- **Rutas y navegación:** nueva sección sobre `AuthGuard` y `/export-runner`.
- **Estructura de módulos:** enlaces a las páginas del layout.

---

## [2026-10-09] — Header: buscador de guía de transportadora ("Consultar guía")

### Nuevas funcionalidades

- **Botón "Consultar guía"** en el header, antes de los avisos y separado por una línea.
  Abre el nuevo modal `ConsultaGuiaModalComponent`: el usuario escribe o pega el número
  de la guía y ve los datos del pedido (transportadora, tienda, estado con color y fecha),
  el link del PDF con **Abrir** y **Copiar**, y la vista previa del PDF en computador.
- Búsqueda por `Numeropreenvio` en `PedidosInter` con el índice `Numeropreenvio_1`,
  pidiendo solo los 8 campos que usa el modal: **113 ms** y **~1 KB** de respuesta.
- **Solo guías de mis tiendas:** la búsqueda se filtra por `tiendas_asignadas`, como en
  la pantalla de Pedidos. Si la guía es de otra tienda, una segunda consulta que solo pide
  el número confirma que existe y se muestra "Esta guía no pertenece a tus tiendas", sin
  ningún dato de ella. Los usuarios con `Todas` ven cualquier guía.
- **Recientes:** las últimas 5 guías encontradas, por usuario y solo durante la sesión
  (`sessionStorage`), con botón Limpiar.
- Mensajes distintos para guía inexistente, guía sin PDF, guía de otra tienda, búsqueda
  que tarda (504 / `TimeoutError`) y tiendas todavía sin cargar.

### Cambios

- Enter busca y, con la guía ya mostrada, un segundo Enter abre el PDF.
- El número se limpia de espacios, saltos de línea y guiones antes de buscar.
- La vista previa del PDF no se muestra en celular (Chrome de Android no tiene visor) ni en
  pantallas táctiles angostas; queda el botón Abrir.
- Accesibilidad: Escape cierra, el foco no se sale del modal con Tab y se respeta
  `prefers-reduced-motion`.

### Documentación

- Nueva página **Frontend → Componentes → Consultar Guía (modal del header)**, con el
  flujo, los estados, el filtro por tiendas (y su nota de seguridad), Recientes, la vista
  previa y las 31 pruebas unitarias.
- Agregada una referencia al modal en **Logística → Consultar Guía**.

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

