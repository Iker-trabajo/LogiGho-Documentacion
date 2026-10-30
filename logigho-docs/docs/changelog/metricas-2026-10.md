# Liberación: Métricas de entrega por ciudad y transportadora (octubre 2026)

**Estado:** backend en **producción** desde 2026-10-30 (primera corrida `TODO` verificada). Scheduler y ajuste visual de prod **apagados** a la espera de la revisión de gerencia; despliegue del front a producción por confirmar.

Documentación técnica: [Métricas (visión general)](../backend/metricas/index.md) · [API](../api/metricas.md) · [Entregas](../frontend/views/metricas-efectividad-ciudad.md) · [Panel de administración](../frontend/views/metricas-configuracion-entregas.md).

## Nuevas funcionalidades

- **`ApiLambdaMetricasEntregaCiudad`** (.NET 10): una corrida (domingo 8:00 o manual) lee `PedidosInter` una vez y deja un resumen por mes × ciudad × transportadora. Ver el porcentaje de entrega por ciudad ya no consulta la base grande.
- **Vista Entregas** (Mi Tienda): mejor transportadora por ciudad, listas de mejores ciudades por transportadora, tabla nacional con filtros, semáforo y exportación a Excel.
- **Panel de administración:** parámetros de cálculo (alcance, muestra mínima con ayuda, clasificación de estados), corridas con auditoría y vista previa del ajuste.
- **Ajuste visual del porcentaje** (spec 005): gerencia define un mínimo y el sitio eleva los porcentajes que muestra, sin cambiar el orden ni tocar los datos reales.
- **Infraestructura como código** (45 recursos en prod) y tres pipelines (`CI`, `CD-PreProd`, `CD-Prod` con aprobación).

## Cambios que se notan

- **Un único cálculo, siempre real:** la base y el backend guardan el porcentaje real; el ajuste solo ocurre al mostrar.
- **Archivada se resuelve por fecha** (entrega con solo fecha de entrega, devolución con solo fecha de devolución) y no se puede escoger como entregada o devuelta.
- **TCC oculto** hasta que sus guías estén en `PedidosInter`. XCargo (`D2`) no cuenta.
- **Selector de período retirado:** el alcance lo define la corrida, no el usuario.

## Correcciones durante la validación

- **Firma del JWT sin validar (seguridad):** la primera versión solo decodificaba el token; las rutas van sin authorizer, así que nadie validaba la firma. Se corrigió con `VerificadorJwtCognito` antes de producción.
- **La corrida manual nunca se ejecutaba** (encontrada en PreProd con CloudWatch y Atlas):
    - La autoinvocación serializaba el evento como `Tipo` y el handler solo reconocía `tipo` (el del Scheduler): lo trataba como una petición de API vacía y la corrida quedaba `EN_CURSO`. Se corrigió con `DetectorEventoCorrida` (sin distinguir mayúsculas).
    - Aun reconocida, la manual quedaba `OMITIDA`: tomaba el candado al lanzarse y su autoinvocación lo volvía a tomar y lo veía ocupado. El filtro acepta la re-toma con el mismo `CorridaId`.
    - Los tests no lo habían detectado porque el doble del repositorio modelaba el candado más simple que el real; ahora lo imita.
- **Corridas caídas bloqueaban el panel:** una `EN_CURSO` de más de 20 minutos ahora se muestra como CAÍDA y no deshabilita *Lanzar corrida*.
- **Id de corrida vacío y % de ciudad sin leer** en el panel (nombres de campo del adaptador).
- **`apiUrlRest` de producción** apuntaba al gateway de PreProd (`environment.ts`).
- **Estados disponibles:** el panel solo mostraba los ya configurados; ahora lista todos los observados en la última corrida, con buscador y "Ver más".
- **Nombres en el panel:** "Actualizado por" y "Solicitada por" mostraban el id de Cognito; ahora el backend los resuelve al nombre del usuario (`Users`).
- **Aviso del ajuste:** decía "aplica desde la próxima corrida", pero el ajuste es solo del front y es inmediato.
- **Hover:** el `title` del navegador tardaba y no se parecía a la UI; se reemplazó por un tooltip propio e instantáneo.
- **Entregas abre en Medellín** y el panel de la ciudad vuelve al ranking comparativo de transportadoras.

## Producción: qué se hizo y qué se verificó (2026-10-30)

| Paso | Resultado |
|---|---|
| `terraform apply` `environments/prod/metricas` | **45 recursos**, 0 cambios, 0 destrucciones. Función `Active` en la VPC de prod, timeout 900 s. Scheduler `DISABLED` |
| CD de prod | Ensayo en verde (`#20261030.1`); luego `soloEnsayo: false` |
| Publicación del stage `Produccion` | A mano con `create-deployment`; el humo dio 403 hasta que propagó y luego **401 `SIN_TOKEN`** |
| Base | Configuración inicial, índices y módulos 169/170 insertados. La colección del resumen se había creado con otro nombre y se renombró antes de correr |
| **Primera corrida `TODO`** | **926 476** guías leídas en **1 min 53 s**; 464 181 entregas y 363 769 devoluciones; **98,1 %** con ciudad resuelta; 25 023 documentos; 1 118 ciudades; meses 2024-11 a 2026-10 |

Porcentajes reales por transportadora en esa corrida: Interrapidísimo 56,6 %, Servientrega 50,6 %, Envía 48,0 %, D2E 39,2 %.

**Validación contra la base (solo lectura):**

- Entregas: `Pagada` 452 243 + `Entregada` 1 782 + `Archivada` 10 678, menos 462 de XCargo (excluida) y 60 `Archivada` sin fecha = **464 181**, igual que la lambda.
- La diferencia entre el total de la corrida y la suma por ciudad (8 875 entregas + 6 497 devoluciones) es **15 372**, igual al motivo `SIN_CIUDAD` de la auditoría.
- Devoluciones: la lambda da 1 082 más que el conteo simple de estados porque unifica variantes de texto (p. ej. "Devolución completada" con minúscula). No se comprobó por separado.

Criterios de la spec 004 cumplidos con esta evidencia: CA-10 (≥ 95 % de ciudades resueltas: 98,1 %), CA-11 (corrida `TODO` en menos de 10 min: 1 min 53 s).

## Pendiente

- Activar el **Scheduler** de prod (`ENABLED` en Terraform) y el **ajuste visual** después de la revisión de gerencia.
- Desplegar el **front a producción** (rama de Angular con las vistas y la corrección de `environment.ts`).
- Decidir `Facturado`, `INDEMNIZADA` y `EN PROCESO DE INDEMNIZACION` (siguen como "no cuenta").
- Mejoras en `backlog.md`: un solo gateway y deployment del stage (M-130/M-131), índice en `FechaCarga` (M-124), nota visible del ajuste, que Pedidos lea los parámetros de *Priorizar*.
- Verificar con una llamada autenticada en prod que la lambda alcanza Cognito desde la VPC (la subred es la de inventario, que ya valida JWT).

## Documentación

- Páginas nuevas: seguridad, ajuste visual, panel de administración y esta entrada.
- Páginas actualizadas: visión general, arquitectura, reglas, operación, API, vista Entregas y guía de módulos (campos `estado` y `roles` según lo observado).
