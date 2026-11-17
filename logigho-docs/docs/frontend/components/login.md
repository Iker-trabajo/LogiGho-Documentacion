---

## Autor: Yadilton Torres

Fecha creacion: 2026-11-06
Estado: desarrollo
Tipo: componente

# Componente: LoginComponent, recuperación y cambio de contraseña

**Selector:** `app-login`, `app-recuperar-contrasena`, `app-cambio-contrasena`  
**Ubicación:** `src/app/components/login`, `src/app/components/login/recuperar-contrasena`, `src/app/components/cambio-contrasena`  
**Acceso:** Público (login y recuperar) · Autenticado (cambio desde el perfil)

---

## ¿Qué hace?

- **Iniciar sesión:** sobre el mismo mapa "La Red" del [registro](../views/pages/registro/registro.md), con su estilo.
- **Recuperar la contraseña sin sesión:** "¿Olvidaste tu contraseña?".
- **Cambiar la contraseña desde el perfil:** usa el mismo flujo de recuperación.

Desde el 2026-11-03 el cambio de contraseña ya no pide la contraseña actual. El factor es el código que llega al correo ([decisión del API](../../api/autenticacion.md)). El front ya no llama a `/cambio-pass` ni `/cambio-pass/codigo`.

---

## Rutas

| Ruta | Componente | Parámetros de URL |
| --- | --- | --- |
| `/login` (publicado: `/app/login`) | `LoginComponent` | — |
| `/recuperar` (publicado: `/app/recuperar`) | `LoginComponent` con `RecuperarContrasenaComponent` | `solicitud`: id del enlace del correo; se guarda en memoria y se borra de la barra |

---

## Servicios y endpoints

| Servicio | Método | Endpoint | Cuándo |
| --- | --- | --- | --- |
| `AuthenticateService` | `login()` | Cognito directo (SRP) | Botón "Iniciar sesión" |
| `RecuperacionRepository` | `solicitar(email)` | `POST autenticacion/recuperacion/solicitudes` | Pedir o reenviar el código |
| `RecuperacionRepository` | `confirmar(solicitudId, codigo, contrasena)` | `POST autenticacion/recuperacion/confirmaciones` | Poner la contraseña nueva |

`RecuperacionRepository` usa `insertarGenericoSinAuditoria` y convierte los errores en `ErrorRecuperacion` con el mensaje de cada `Codigo`.

---

## Flujo del inicio de sesión

```
login()
  -> aviso "Verificando tus datos"
  -> AuthenticateService.login(correo, contraseña)
  -> esperarCamion(): espera a que el camioncito del botón termine su viaje (1,5 s por viaje)
  -> correcto: aviso "¡Bienvenido de vuelta!", la tarjeta se abre como puerta,
               la pantalla se funde a oscuro y navega a /app/muro-compartido (con recarga)
  -> incorrecto: el mapa parpadea en rojo, la tarjeta tiembla, mensaje bajo los campos
                 y aviso rojo "No pudimos iniciar sesión" (4 s)
```

**Ajustes de esta versión:**
- **Camioncito:** la respuesta nunca corta el viaje a la mitad, porque `esperarCamion()` redondea al siguiente viaje completo. Con "reducir movimiento" no espera.
- **Sin ventanas emergentes:** se quitó SweetAlert. El acceso denegado a un módulo ahora sale como aviso: "Esta sección no está en tu plan".
- **Sin redes sociales:** se quitaron los botones de Google y Facebook.

---

## Flujo de recuperar contraseña

| Paso | Pantalla | Qué hace |
|---|---|---|
| `correo` | Recupera tu contraseña | Pide el correo y llama a `solicitar()`. Responde igual exista o no la cuenta |
| `revisa` | Revisa tu correo | "Reenviar código" con 60 s de espera |
| `codigo` | Crea tu contraseña nueva | Se abre desde el enlace `?solicitud=`. Seis casillas reales para el código (escribir, pegar, borrar), contraseña con requisitos en vivo y confirmación |
| `listo` | ¡Contraseña actualizada! | Cierra la sesión local, porque el servidor cerró todas, y vuelve al ingreso |

**Errores:**
- **Código malo:** se puede reintentar.
- **Código ya usado, vencido o con demasiados intentos:** solo deja "Pedir otro código".

## Cambio desde el perfil

`CambioContrasenaComponent` (modal) solo envía el código al correo de la sesión y lo muestra enmascarado (`an***@dominio`). La contraseña nueva se pone desde el enlace del correo, en la misma pantalla de recuperar.

---

## Celular

- **Barra de arriba:** respeta la muesca del iPhone.
- **Botones:** a todo lo ancho y con área de toque grande.
- **Sin efectos pegados:** en pantallas táctiles no se quedan los efectos de pasar el mouse.
- **Autocompletado:** los campos conservan el fondo oscuro.
- **Casillas del código:** caben desde 320 px.

---

## Pruebas unitarias

| Archivo | Qué cubre |
|---|---|
| `login.component.spec.ts` | Espera del camioncito (rápido, lento, sin animación), error después del camión y que no repita el ingreso |
| `recuperar-contrasena.component.spec.ts` | Pedir código, confirmar, errores y las casillas (escribir, pegar, borrar) |
| `cambio-contrasena.component.spec.ts` | Envío del código desde el perfil |

---

## Historial de cambios

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-11-04 | Yadilton Torres | Recuperar y cambiar contraseña migrados a las rutas de recuperación; el login sin botones de redes sociales. |
| 2026-11-06 | Yadilton Torres | Casillas reales del código, avisos en vez de SweetAlert, espera del camioncito, puerta de bodega más fluida, ajustes para celular y pruebas unitarias. |

---

## Observaciones

- **Recarga al entrar:** al entrar se recarga la página (`window.location.reload()`). Se disimula con el fundido, pero quitarla haría el ingreso más fluido; hay que revisar antes el menú.
- **Botón "Regístrate":** navega a `app/registro`. En el servidor local manda a otra pantalla; confirmar cómo se comporta publicado.
- **Cuentas creadas con Google/Facebook:** no tienen contraseña propia y reciben `500` al confirmar la recuperación (M-229). Falta acordar el mensaje.
