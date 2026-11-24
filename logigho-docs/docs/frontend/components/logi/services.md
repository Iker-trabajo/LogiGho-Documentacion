---
autor: Yadilton Torres
fecha_creacion: 2026-11-24
ultima_actualizacion: 2026-11-24
estado: desarrollo
nivel: 2
---

# Logi: servicios

**Autor:** Yadilton Torres

**Ubicación:** `SitioLogiGho/src/app/components/logi-pet/services/`

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

## `LogiEntornosService` (`logi-entornos.service.ts`)

«Cambiar entorno» (experiencias). Contrato: [Preferencias de usuario](../../../api/preferencias.md).

| Elemento | Qué hace |
|---|---|
| `cargar(forzar?)` | `GET /preferencias/mia`, una vez por sesión. Nunca lanza: si falla, el usuario sigue en `clasica` |
| `cambiar(id, confirmado?)` | `PUT`; cambia solo si el servidor confirma |
| `cambiarTema(tema)` | Tema dentro del entorno activo |
| `entornos`, `activo`, `temas`, `hayParaElegir` | Señales para la lista de Logi (la opción sale con 2 o más entornos) |
| `registrarGuardia(fn)` / `hayCambiosSinGuardar()` | Una pantalla avisa cambios sin guardar; si la guardia falla, se toma como sin cambios |

Con `environment.preferenciasActivas = false` (producción) el estado es `apagado`, no llama al servidor y solo existe «Clásica».

---

## Pruebas

- `logi.service.spec.ts`: avisos sin Logi, una sola espera con varias peticiones, conexión tardía, desconexión, error del oyente sin romper la app, mensaje del servidor, frase por código, versión nueva y detalle técnico.
- `logi-entornos.service.spec.ts`: apagado sin llamar al servidor, guardia que se registra y se quita, guardia que falla.
