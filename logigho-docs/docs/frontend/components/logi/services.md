---
autor: Yadilton Torres
fecha_creacion: 2026-11-24
ultima_actualizacion: 2026-11-24
estado: desarrollo
nivel: 2
---

# Logi: servicios

**Autor:** Yadilton Torres

**Ubicación:** `SitioLogiGho/src/app/components/logi-assistant/services/`

---

## `LogiService` (`logi.service.ts`)

Puente entre la plataforma y Logi (`providedIn: 'root'`). Cualquier parte de la app lo inyecta para que Logi reaccione sin conocer el componente. Si Logi no está en pantalla (login, landing), los avisos se ignoran.

| Método | Quién lo llama | Qué pasa |
|---|---|---|
| `conectar(oyente)` | El componente al iniciar | Devuelve la función para desconectarse. Si hay una petición lenta en curso, empieza a esperar de una vez |
| `peticionLenta()` / `peticionLentaTermino(ok)` | `logi-espera.interceptor` (más de 3 s) | Avisa la espera **una sola vez** aunque haya varias, y termina cuando no queda ninguna |
| `peticionFallo(error)` | El interceptor | Mensaje del servidor si es legible (`mensaje`, `message`…), o una frase según el código HTTP; más el detalle técnico |
| `falloCodigo(error)` | `logi-error.handler` | Versión nueva (recargar), red cortada, sin espacio, o «Algo falló dentro de esta pantalla» |
| `avisar(texto)` | Cualquiera (ej. campana) | Logi dice el mensaje |

Todo corre fuera de la zona de Angular y dentro de `try/catch`: **Logi nunca rompe el flujo de la app**.

Funciones exportadas: `motivoDeFallaCodigo()`, `tecnicoDeServidor()` (estado, servicio sin parámetros, detalle y referencia de `traceId`, `requestId` o `x-amzn-RequestId`) y `tecnicoDeCodigo()` (nombre, mensaje y dos líneas de la traza).

---

## `LogiExperienciasService` (`logi-experiencias.service.ts`)

«Cambiar entorno» (experiencias). Contrato: [Preferencias de usuario](../../../api/preferencias.md).

Desde la integración del 2026-11-25 **no hace peticiones propias**: delega en el `ExperienciaActivaStore` del
[motor de experiencias](../../experiencias/index.md), el mismo que usa el selector del header. Así Logi y el selector comparten estado y versión
(antes cada uno tenía la suya y chocaban con 409).

| Elemento | Qué hace |
|---|---|
| `cargar(forzar?)` | Pide al store que lea `GET /preferencias/mia` (`iniciar()`); `forzar` es «Reintentar». Nunca lanza: si falla, el usuario sigue en `clasica` |
| `cambiar(id, confirmado?)` | Revisa guardia, `ya-activo` y `en-curso`; baja el shell de la experiencia **antes** de guardar (si no carga: `carga-fallida`); luego `store.cambiar(id)` (`PUT`, solo cambia si el servidor confirma). Traduce el aviso del store a un motivo de Logi y lo descarta para no mostrar dos mensajes |
| `cambiarTema(tema)` | Hoy devuelve `desconocido`: cada experiencia tiene un solo tema |
| `experiencias`, `activo`, `hayParaElegir` | Salen de `store.opciones()`, `store.activa()` y `store.hayVarias()`; el estado (general, beta, borrador) es el que da el servidor (`store.estadosServidor`). La opción sale con 2 o más experiencias |
| `estado` | `apagado` (bandera apagada), `sin-cargar`, `cargando`, `fallo` (`store.lecturaFallida`) o `listo` |
| `registrarGuardia(fn)` / `hayCambiosSinGuardar()` | Una pantalla avisa cambios sin guardar; si la guardia falla, se toma como sin cambios |

Con `environment.preferenciasActivas = false` (producción) el estado es `apagado`, no llama al servidor y Logi muestra «Muy pronto».

!!! info "Dónde se monta Logi"
    Desde la misma integración Logi se monta **una sola vez** en `ShellHostComponent` (fuera del `ngComponentOutlet`), no dentro del layout clásico.
    Así aparece en Clásica y en Aurora y no se destruye al cambiar de experiencia (el mensaje «¡Listo! Ya estás en Aurora» sobrevive al cambio).
    Recibe el menú de `PlataformaFacade.menu()`. En Aurora móvil sube sobre la barra inferior (regla en `experiencias/aurora/estilos/_global.scss`).

---

## Pruebas

- `logi.service.spec.ts`: avisos sin Logi, una sola espera con varias peticiones, conexión tardía, desconexión, error del oyente sin romper la app, mensaje del servidor, frase por código, versión nueva y detalle técnico.
- `logi-experiencias.service.spec.ts`: delegación en el store (con un `PREFERENCIAS_PORT` falso), estados, cargar, cambiar, errores 422 y 500, guardia y `en-curso`.
