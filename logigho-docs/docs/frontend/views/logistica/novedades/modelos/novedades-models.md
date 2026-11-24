## Autor: Yadilton Torres

Fecha creacion: 2026-10-30

Estado: desarrollo

## Modelos: novedades.models.ts

**Ubicación:** `src/app/views/logistica/novedades/models/novedades.models.ts`

---

## Transportadoras

```typescript
type Transportadora = 'INTERRAPIDISIMO' | 'ENVIA' | 'SERVIENTREGA';
```

`TRANSPORTADORAS` define el nombre visible y el logo de cada una (`assets/images/logos/actuales/`). Es la única lista: el selector, la tabla, el panel y el Excel la leen de aquí.

---

## `NovedadFila` — una guía en seguimiento

Una sola forma para las tres transportadoras, aunque vengan de colecciones distintas. Así filtros, KPIs, tabla y Excel no necesitan saber de dónde vino cada guía.

| Campo | Tipo | De dónde sale |
| ----- | ---- | ------------- |
| `guia` | `string` | Inter: `Guia`; Envía/Servientrega: `Numeropreenvio` |
| `transportadora` | `Transportadora` | — |
| `tienda`, `idTienda` | `string` | `Tienda`, `Idtienda` |
| `estadoSeguimiento` | `'Activo' \| 'Cerrado'` | Inter: `EstadoSeguimiento`; Envía/Servientrega siempre `Activo` |
| `etapa?` | `'novedad' \| 'devolucion'` | Servientrega con mensaje "Devolucion:" = `devolucion`; sin valor = `novedad` |
| `prioridad` | `'Normal' \| 'Alta' \| null` | Solo Inter (`Prioridad`, la pone el robot) |
| `estadoInter` | `string` | Inter: `UltimaRespuesta.estado_actual`; Envía: `Estado` sin la ciudad; Servientrega en devolución: `EN DEVOLUCIÓN` |
| `motivo` | `string \| null` | Solo Inter: `UltimoMotivo.Motivo` tipificado |
| `descripcion` | `string \| null` | Envía/Servientrega: `Descripcion Asociada` |
| `urlGuia` | `string \| null` | PDF de la guía (`PedidosInter.UrlGuia`) |
| `ciudad` | `string` | Inter: `UltimoMotivo.Ciudad`; Envía: sacada de `Estado` ("EN NOVEDAD POR MEDELLIN") |
| `fechaNovedad`, `fechaCreacion` | ISO | Inter: `UltimoMotivo.Fecha`, `FechaCreacion`; Envía/Servientrega: `Fecha de última actualización`, `FechaCarga` |
| `consultas`, `consultaFallida` | — | Solo Inter: `ConsultasTotales`, `Solicitud.Estado` |
| `intentosEntrega?` | `number \| null` | `PedidosInter.IntentosEntrega` (Inter se cruza por `Numeropreenvio`) |
| `tipoEntrega`, `mensajero` | `string` | Inter: `UltimaRespuesta.envio.tipo_entrega`, `UltimoMotivo.Mensajero`; Envía: `TipoEntrega` (1 = dirección, 2 = oficina) |
| `geoUrl` | `string \| null` | Inter: `UltimoMotivo.Geo` (último intento) |
| `destinatario` | `{ nombre, telefono, direccion }` | Inter: `UltimaRespuesta.destinatario` (al abrir el detalle); Envía/Servientrega: el propio pedido |
| `historial` | lista | Inter: `Historial[].Motivo` — motivo, estado, fecha, ciudad y `geoUrl` de cada intento |
| `flujo` | `PasoFlujo[]` | Inter: `UltimaRespuesta.flujo` (trazabilidad) |
| `evidencias?` | `Evidencia[]` | Inter: `UltimaRespuesta.gestion_app` (fotos en S3) |
| `detalleCargado?` | `boolean` | `false` en Inter hasta abrir el detalle |

---

## `Evidencia` y `FotoEvidencia`

```typescript
interface Evidencia { s3Key: string; fecha: string | null; tipo: string; descripcion: string; orden: number; }
type FotoEvidencia = { estado: 'cargando' } | { estado: 'lista'; url: string } | { estado: 'error' };
```

`s3Key` es la ruta exacta en el bucket `wsinter-evidencias-prod`, por ejemplo `gestion-app/240064484213/2026-10-30/240064484213-1.jpg`. La guarda el robot de Inter en `UltimaRespuesta.gestion_app[].s3_key`, así que no hay que listar carpetas ni adivinar fechas.

---

## Otros tipos

| Tipo | Valores |
| ---- | ------- |
| `PasoFlujo` | `{ estado, ciudad, fecha, motivo }` — un paso de la trazabilidad de Inter |
| `EstadoCarga` | `'cargando' \| 'error' \| 'listo'` |
| `TonoMotivo` | `'rehusado' \| 'ausente' \| 'direccion' \| 'otro'` — color del chip del motivo |
