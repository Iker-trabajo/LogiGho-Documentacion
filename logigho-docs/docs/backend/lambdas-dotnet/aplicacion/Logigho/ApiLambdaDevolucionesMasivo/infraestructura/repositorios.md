## Autor: Iker Acevedo

Fecha creacion: 2026-08-26

Estado: produccion

## Infraestructura: Repositorios

**Ubicación:** `Infraestructura/Repositorio/`

---

## `JobRepository` — persistencia del job

**Colección:** `JobsDevolucion`. Único canal entre Iniciador, Worker y front.

| Método | Qué hace |
| ------ | -------- |
| `CrearAsync(job)` | `InsertOneAsync` |
| `ObtenerAsync(jobId)` | `findOne` por `_id` |
| `ActualizarProgresoAsync(jobId, rechazadas, resueltas, yaProcesadas, pendientes)` | `PushEach` sobre `Rechazadas`, `Inc` sobre los 3 contadores, `Set` de `GuiasPendientes` y `FechaUltimaActividad` |
| `IncrementarContinuacionesAsync(jobId)` | `Inc(Continuaciones, 1)` |
| `MarcarCompletadoAsync(jobId)` | `Set Estado=Completado, GuiasPendientes=[], FechaFin=ahora` |
| `MarcarFallidoAsync(jobId, mensaje)` | `Set Estado=Fallido, MensajeError, FechaFin=ahora` |
| `ObtenerJobsAbiertosAsync(usuario)` | `Estado in [Pendiente, EnProceso]`, proyecta excluyendo `Rechazadas`, limit 10 |

**`RegistrarMapeo()` (constructor estático, corre una vez por contenedor):**

```csharp
ConventionRegistry.Register("EnumsComoTexto", EnumRepresentationConvention(BsonType.String));
BsonClassMap.RegisterClassMap<JobDevolucion>(m => m.MapIdMember(j => j.JobId));
BsonClassMap.RegisterClassMap<GuiaProcesada>(m => {
    m.UnmapMember(g => g.FueResuelta);
    m.UnmapMember(g => g.RequiereActualizarInventario);
    m.UnmapMember(g => g.MereceReintentoConOtraTransportadora);
    m.UnmapMember(g => g.EsRecienResuelta);
});
```

Los enums se guardan **como texto** ("PedidoAnulado"), no como número — si mañana alguien reordena `MotivoRechazo`, los documentos ya guardados seguirían diciendo lo correcto en vez de convertirse silenciosamente en otro motivo. Además hace la colección legible desde Compass.

**Actualizar solo `Rechazadas`, nunca todas las guías:** las resueltas solo suman a un contador (`Inc`) — su detalle completo vive en `InventarioDevolucion`. Guardarlo dos veces haría crecer el documento sin techo con archivos grandes. Ver [ADR-002](../adr/ADR-002-jobdevolucion-solo-contadores.md).

---

## `DevolucionRepository` — acceso a `PedidosInter` e `InventarioDevolucion`

Dos reglas de datos que hay que respetar siempre en `PedidosInter`:

1. **`Numeropreenvio` está guardado a veces como string y a veces como entero.** Todo filtro sobre ese campo lleva las dos formas (`FiltroPorCampo` arma un `$or` con ambas), o pierde resultados en silencio.
2. **Envía entrega sus guías con ceros a la izquierda** (`034056692586`) pero en `PedidosInter` viven como entero (`34056692586`). Parsear a `long` descarta el cero por sí solo, que es justo lo que hace falta.

| Método | Qué hace |
| ------ | -------- |
| `ObtenerGuiasYaProcesadasAsync(guias)` | Filtra `InventarioDevolucion` por `Guia` + `Validacion=="OK"` — solo cuenta como procesada lo que se resolvió con éxito |
| `BuscarPorGuiaDevolucionAsync(guias)` | Busca en `PedidosInter` por `GuiaDevolucion` |
| `BuscarPorNumeropreenvioAsync(guias)` | Busca por `Numeropreenvio` |
| `MarcarDevolucionCompletadaAsync(resueltas)` | `BulkWrite` de `UpdateOne` — `Estado="Devolucion Completada"`, `"Fecha Dev Completada"` (hora Colombia, excepción deliberada — ver [ADR-004](../adr/ADR-004-fechas-utc.md)), `"Fecha Devolucion"`, `GuiaDevolucion` |
| `RegistrarEnInventarioDevolucionAsync(procesadas, usuarioEmail, jobId)` | `InsertMany` en `InventarioDevolucion` — todas menos `YaProcesada`. Incluye `JobId` (ver [ADR-003](../adr/ADR-003-jobid-en-inventariodevolucion.md)) |

**Idempotencia filtra por `Validacion=="OK"` específicamente:** una guía **rechazada** no bloquea el reintento — si el operario corrige el motivo (le asignan la tienda, por ejemplo) y vuelve a escanear la misma guía, tiene que poder reprocesarse.

**Tope de 500 por `$in`** (`TamanoTrozo`): un solo filtro con miles de valores genera un documento de consulta enorme y presiona la memoria del servidor de Mongo — se parte en trozos.

---

## `UsuarioRepository` — tiendas asignadas

**Colección:** `Users` (con mayúscula).

```
ObtenerContextoAsync(identidad)
  filtro = email == identidad.Email  OR  cognitoId == identidad.Sub
  (cognitoId es un arreglo en Users — un Eq contra un arreglo hace match si ALGUN elemento coincide, no hace falta $in)
  si no encuentra documento o "tiendas" queda vacio -> null
  devuelve ContextoUsuario { Email = el de la BASE (no el del token), TiendasAsignadas }

ObtenerContextoAsync(email)   sobrecarga que usa el Worker (no tiene token, invocacion asincrona)
  delega en la version completa con IdentidadUsuario(email, null)
```

**Por qué busca por email O por sub:** el pool de Cognito no garantiza que el JWT incluya el claim `email` — buscar también por `sub` (contra `cognitoId`) hace que funcione sin depender de esa configuración.

**Por qué el email real sale de la base y no del token:** es el que se guarda como `Usuario` del job y en `InventarioDevolucion` — si el token no lo trajo, igual está disponible porque se pidió en la proyección de Mongo.

`LeerTiendas` acepta el campo `tiendas` tanto si viene como arreglo BSON como si viene como texto separado por comas — tolera las dos formas en que distintos flujos de administración pudieron haber escrito el documento.

---

## `MongoConexion` — cliente único por contenedor

```csharp
MaxConnectionPoolSize = 10
MinConnectionPoolSize = 0
MaxConnectionIdleTime = 1 minuto
MaxConnectionLifeTime = 30 minutos
```

El pool acotado es la pieza central del proyecto que dio origen a este módulo: la mayoría de repositorios del ecosistema LogiGho crean el cliente Mongo **sin** `MaxConnectionPoolSize`, y el driver toma 100 por defecto. Con decenas de contenedores Lambda concurrentes en hora pico, eso explica los picos de conexiones vistos en CloudWatch que motivaron reescribir este módulo desde cero (99% de CPU en DocumentDB).
