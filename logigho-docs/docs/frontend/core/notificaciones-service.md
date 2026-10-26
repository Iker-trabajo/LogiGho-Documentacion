---

## Autor: Yadilton Torres

Fecha creacion: 2026-10-21
Estado: desarrollo
Tipo: servicio

# Servicio: NotificacionesService

**Ubicación:** `src/app/core/application/services/notificaciones/`
**Scope:** `providedIn: 'root'`

---

## ¿Qué hace?

Maneja las notificaciones de la plataforma que se ven en la [campana del header](../components/campana-notificaciones.md): las consulta cada cierto tiempo, las envía, cambia su estado (NUEVA → LEIDA → HECHA), le envía la respuesta a quien la creó y trae el historial.

Hoy lo usa la pantalla de [Exportación de datos](../views/gestion-datos/export-data/export-data.md): cuando el usuario necesita exportar más del límite, le escribe al equipo de desarrollo y el mensaje le llega a su campana. Cuando el equipo responde, la respuesta le llega a la campana de quien lo envió.

---

## Archivos

| Archivo | Responsabilidad |
| --- | --- |
| `notificacion.models.ts` | Tipos, estados, interfaz `Notificacion` y configuración por tipo (`TIPOS_NOTIFICACION`) |
| `notificaciones.repository.ts` | Llamadas HTTP a la colección `NotificacionesPlataforma` (consultar, insertar, cambiar estado). No toca la vista |
| `notificaciones.service.ts` | Estado con signals, revisión periódica, enviar, marcar leída, marcar hecha, responder e historial |
| `aviso-notificacion.service.ts` | Aviso arriba a la derecha al enviar, responder o fallar una notificación |

---

## Colección `NotificacionesPlataforma`

> La colección `Notificaciones` ya existe y es de la red social (Muro compartido). Por eso esta tiene nombre propio.

| Campo | Tipo | Descripción |
| --- | --- | --- |
| `Tipo` | `string` | `SOLICITUD_EXPORTACION` o `SOLICITUD_RESUELTA` |
| `Mensaje` | `string` | Texto que escribió el usuario (en una respuesta, la respuesta) |
| `Contexto` | `object` | Datos de la pantalla desde donde se envió, se llenan solos. En Exportación: `Coleccion`, `Formato`, `Rango`, `Registros`, `Columnas`, `Limite` |
| `CreadoPor` | `string` | Correo de quien la envió (`sessionStorage['emailaddress']`) |
| `Roles` | `string[]` | Roles que la ven; vacío si va a personas |
| `Usuarios` | `string[]` | Correos que la ven; vacío si va a roles |
| `Estado` | `string` | `NUEVA` al enviarse, `LEIDA` cuando alguien la abre, `HECHA` cuando la responden o la marcan como vista |
| `FechaCreacion` | `string` | Fecha ISO |
| `IdOrigen` | `string` | En una respuesta, el `_id` de la notificación que se responde |
| `RespondidaPor` | `string` | Correo de quien respondió |
| `Respuesta` | `string` | Mensaje de respuesta para quien la envió |
| `FechaRealizada` | `string` | Cuándo se atendió (fecha ISO) |

El estado es común para todos los que la reciben: si un desarrollador la abre, queda `LEIDA` para todos.

**Índices sugeridos** (los crea el backend): `{ Roles: 1, Estado: 1, FechaCreacion: -1 }` y `{ Usuarios: 1, Estado: 1, FechaCreacion: -1 }`.

---

## Tipos de notificación

Se configuran en `TIPOS_NOTIFICACION`. Para que un tipo le llegue a otros roles se cambia ahí.

| Tipo | Etiqueta | Icono / tono | Le llega a | Acción | Al atenderla |
| --- | --- | --- | --- | --- | --- |
| `SOLICITUD_EXPORTACION` | Solicitud de exportación | `cilCloudDownload` / naranja | Rol `Desarrollador` | `responder` (respuesta obligatoria) | Queda `HECHA` y se crea una `SOLICITUD_RESUELTA` para quien la envió |
| `SOLICITUD_RESUELTA` | Tu solicitud fue atendida | `cilTask` / verde | El correo de quien hizo la solicitud | `entendido` | Solo pasa al historial |

Un tipo desconocido se muestra como "Notificación" que solo se puede quitar.

---

## Signals expuestos

| Signal | Tipo | Descripción |
| --- | --- | --- |
| `notificaciones` | `Notificacion[]` | Pendientes (`NUEVA` y `LEIDA`): primero las nuevas, cada grupo de la más nueva a la más vieja |
| `historial` | `Notificacion[]` | Últimas `HECHA` (máximo `LIMITE_HISTORIAL = 20`); se consultan solo al abrir el historial |
| `nuevas` | `computed<number>` | Cuántas están en `NUEVA` (contador de la campana) |
| `cargando` / `cargandoHistorial` | `boolean` | Hay una consulta en curso |
| `ultimaRevision` | `Date \| null` | Cuándo se consultó por última vez |
| `puedeRecibir` | `boolean` | El usuario tiene correo o algún rol que recibe notificaciones |

---

## Métodos

| Método | Qué hace |
| --- | --- |
| `iniciar()` | Empieza a revisar. Espera los roles del usuario (hasta 1 minuto, cada 3 s), consulta ya y luego cada **2 min** si tiene un rol que recibe solicitudes o cada **5 min** si no. Solo revisa con la pestaña visible y vuelve a consultar al regresar a la pestaña. Se puede llamar varias veces: solo arranca una |
| `detener()` | Deja de revisar (al salir del layout) |
| `revisar()` | Consulta las pendientes de los roles y del correo del usuario |
| `revisarHistorial()` | Consulta las últimas 20 `HECHA` |
| `enviar(tipo, mensaje, contexto)` | Crea una notificación `NUEVA` para los roles configurados del tipo |
| `marcarLeida(notificacion)` | `NUEVA` → `LEIDA`. Se ve de inmediato; si falla vuelve a como estaba |
| `marcarHecha(notificacion, respuesta)` | Pasa a `HECHA` (al historial). Ver flujo abajo |

---

## Flujo al marcar como hecha

```
marcarHecha(notificacion, respuesta)
  -> si el tipo es "responder" y no hay respuesta → Error('La respuesta es obligatoria')
  -> la quita de pendientes
  -> repositorio.cambiarEstado(HECHA, { RespondidaPor, Respuesta, FechaRealizada })
       // con "entendido" solo se guarda FechaRealizada (nadie respondió)
       // si falla → vuelve a pendientes y lanza el error
  -> la pone de primera en el historial
  -> si el tipo tiene tipoRespuesta:
       repositorio.insertar({ Tipo: SOLICITUD_RESUELTA, Usuarios: [CreadoPor], IdOrigen, Mensaje: respuesta, ... })
       // si solo falla esto → queda HECHA y lanza RespuestaNoEnviadaError(destinatario)
```

---

## Repositorio: `NotificacionesRepository`

| Método | Endpoint | Descripción |
| --- | --- | --- |
| `consultar(roles, correo, estados, limite?)` | `GET /metodoGenerico?coleccion=NotificacionesPlataforma&Roles=...&Estado=...&mcomp=2&lote=30` y `...&Usuarios=<correo>&lote=10` | El método genérico no hace "o" entre campos: hace una consulta por `Roles` y otra por `Usuarios` en paralelo, las junta sin repetidas y las ordena de la más nueva a la más vieja. Con `limite` (historial) cada consulta pide solo esa cantidad |
| `insertar(notificacion)` | `insertarGenerico` en `metodoGenerico?coleccion=NotificacionesPlataforma` | Guarda una notificación nueva |
| `cambiarEstado(notificacion, estado, extra)` | `actualizarGenericoSinAuditoria` (`LEIDA`) o `actualizarGenerico` (`HECHA`) | Envía el registro completo con el nuevo estado. `LEIDA` va sin auditoría porque no es una acción de negocio; `HECHA` sí queda en `LogAuditoria` |

Las respuestas llegan comprimidas en Zstd. El `_id` puede llegar como texto o como `{ $oid }` y se deja siempre como texto.

### Interruptor de escritura

`NotificacionesRepository.ESCRITURA_ACTIVA` (hoy `true`). En `false`, `insertar` y `cambiarEstado` lanzan `EscrituraDesactivadaError` sin llamar al backend; Exportación muestra "Envío en preparación" y la campana "Por ahora los cambios no se guardan".

---

## Errores

| Error | Cuándo |
| --- | --- |
| `EscrituraDesactivadaError` | Se intenta guardar con `ESCRITURA_ACTIVA = false` |
| `RespuestaNoEnviadaError` | La notificación quedó `HECHA` pero no se pudo crear la respuesta para quien la envió. Trae `destinatario` |

---

## AvisoNotificacionService

`mostrar(tipo, titulo, texto?)` muestra un aviso arriba a la derecha (SweetAlert en modo toast): tarjeta blanca, icono en círculo de color, título, texto, X para cerrar y barra de tiempo. Dura 4,5 s y se pausa con el mouse encima. En celular ocupa todo el ancho.

| Tipo | Color | Uso |
| --- | --- | --- |
| `exito` | Verde | Mensaje enviado, respuesta enviada |
| `error` | Rojo | No se pudo enviar o guardar |
| `info` | Azul | Espera entre mensajes, escritura apagada |
| `advertencia` | Naranja | Quedó hecha pero no se pudo avisar a quien la envió |

Los estilos son globales en `src/scss/_aviso-notificacion.scss` (importado en `styles.scss`) porque SweetAlert pinta fuera de los componentes. Escapa el título y el texto antes de ponerlos en el HTML. El `AvisoToastService` de Pauta no se cambió.

---

## Pruebas

| Archivo | Pruebas |
| --- | --- |
| `notificaciones.service.spec.ts` | 17 |
| `notificaciones.repository.spec.ts` | 8 |
| `aviso-notificacion.service.spec.ts` | 3 |

---

## Historial de cambios

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-10-21 | Yadilton Torres | Creación del servicio, repositorio y modelos de notificaciones (colección `NotificacionesPlataforma`) |
| 2026-10-21 | Yadilton Torres | Respuestas: `SOLICITUD_RESUELTA` le llega a quien envió la solicitud; campos `Usuarios`, `IdOrigen`, `RespondidaPor`, `Respuesta` y `FechaRealizada` |
| 2026-10-21 | Yadilton Torres | Historial con las últimas 20 `HECHA` y `AvisoNotificacionService` |

---

## Observaciones

- **La colección la debe crear el backend** con los índices sugeridos, y el filtro por rol debe hacerse en el servidor: hoy el frontend pide por `Roles` y `Usuarios`, pero cualquiera podría pedir otras desde el navegador.
- **Localhost apunta a producción:** enviar desde localhost crea registros reales.
- **El historial es por fecha de creación:** "últimas 20" se cuenta por `_id`, no por `FechaRealizada`.
- **Los roles se comparan escritos como en la configuración** (ej. `Desarrollador`), porque la base compara mayúsculas exactas.
