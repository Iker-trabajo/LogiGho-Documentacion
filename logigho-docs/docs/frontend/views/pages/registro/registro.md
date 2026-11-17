---

## Autor: Yadilton Torres

Fecha creacion: 2026-11-06
Estado: desarrollo
Tipo: vista

# Vista: RegistroComponent ("La Red")

**Selector:** `app-registro`  
**Ubicación:** `src/app/views/pages/registro`  
**Acceso:** Público

---

## ¿Qué hace?

Es la página pública donde una tienda pide su cuenta de LogiGho sin ayuda de nadie. La persona responde seis preguntas sobre un mapa 3D de Colombia que reacciona a cada respuesta. Al enviar, el API le manda un código al correo. Desde el enlace de ese correo vuelve a la misma vista, escribe el código, crea su contraseña y la cuenta queda activa con el rol `Red social`, hasta que Comercial le da acceso.

La vista tiene dos pantallas según la URL:

| Pantalla | Cuándo se muestra | Clase |
|---|---|---|
| Formulario | Sin `?solicitud=` | `FormView` |
| Código y contraseña | Con `?solicitud=<id>` (enlace del correo) | `ConfirmacionView` |

El mapa, la caja animada y los avisos son comunes a las dos. El contrato del API está en [API de autenticación](../../../../api/autenticacion.md) y la guía general del front en [Autenticación y registro](../../../autenticacion.md).

---

## Ruta

| Ruta | Guard | Parámetros de URL |
| --- | --- | --- |
| `/registro` (publicado: `/app/registro`) | — | `solicitud`: id del enlace del correo. Se lee una vez, se guarda solo en memoria y se borra de la barra con `history.replaceState`. |

La ruta carga el componente de forma diferida (`loadComponent`), así Three.js y los datos del mapa no entran en el bundle inicial.

---

## Estructura de carpetas

```
views/pages/registro/
├── registro.component.ts/html     → monta el formulario o la confirmación, fuera de la zona de Angular
├── models/registro.models.ts      → ContactoRegistro, SolicitudRegistro, ErrorRegistro
├── repository/registro.repository.ts → única puerta al API del registro
├── styles/registro.scss           → estilos de toda "La Red" (también los usa el login)
└── la-red/
    ├── form/      → FormView, ConfirmacionView, StepController, avisos, rutas, textos (copy.es.ts)
    ├── host/      → MapaLaRed, PackageView (caja), choreography, sonido, soporte del navegador
    ├── scene/     → NetworkScene (Three.js) y sus capas: puntos, rutas, camiones, bordes, etiquetas
    ├── data/      → carga de municipios y bordes (assets/la-red/data)
    └── fallback/  → PosterMap: mapa SVG cuando el equipo no aguanta WebGL
```

El formulario y la escena son TypeScript sin Angular y corren con `NgZone.runOutsideAngular`: la animación no dispara la detección de cambios en cada cuadro. El detalle de la escena está en [Mapa "La Red"](la-red-mapa.md).

---

## Servicios y endpoints

Todas las llamadas pasan por `RegistroRepository` y usan `insertarGenericoSinAuditoria`, porque los cuerpos llevan códigos y contraseñas.

| Servicio | Método | Endpoint | Cuándo |
| --- | --- | --- | --- |
| `RegistroRepository` | `enviarSolicitud()` | `POST registerContact` y luego `POST autenticacion/registro/solicitudes` | Al enviar el formulario. Si `registerContact` falla se ignora: el contacto se crea al confirmar |
| `RegistroRepository` | `confirmar()` | `POST autenticacion/registro/confirmaciones` | Al crear la contraseña |
| `RegistroRepository` | `reenviarCodigo()` | `POST autenticacion/registro/solicitudes/reenvio` | Botón "Reenviar código" (60 s de espera) |

Los errores del API se convierten en `ErrorRegistro(codigo, motivos)` y cada pantalla decide el mensaje por el `Codigo`.

---

## Pantalla 1: formulario

Seis preguntas, una por paso (`StepController`), más la revisión final:

| Paso | Pregunta | Campo enviado | Efecto en el mapa |
|---|---|---|---|
| `name` | ¿Cómo te llamas? | `userName` | Aparece el punto de la persona con su nombre |
| `city` | ¿Desde qué ciudad despachas? | `city` | El punto viaja a la ciudad y se encienden las rutas; buscador con sugerencias de los 1.103 municipios |
| `email` | Correo | `email` | Arco de verificación; sugiere el dominio si hay un error de escritura (`gmal.com` → `gmail.com`) |
| `source` | ¿Cómo nos conociste? | `source` | Señal desde el punto |
| `phone` | Teléfono | `phoneNumber` | Onda desde el punto |
| `timing` | ¿Cuándo quieres empezar? | `comienzo` | Intensidad de las rutas según la respuesta |
| `review` | Revisión y autorización de datos | `autorizaDatos: true` | La caja se cierra con cinta |

`isColombian: true` se envía siempre. `NombreComercio`, `ventasPorMes` y `businessType` existen en el contrato pero el formulario no los pregunta.

**Caja animada (`PackageView`):** se arma con cada respuesta (etiqueta, código de barras, sticker, teléfono, sello). Al enviar vuela en arco hasta el punto de la persona (40 fotogramas calculados, sin sombras mientras vuela), los camiones salen por las rutas y en su lugar cae el sello **"Paquete enviado · Revisa tu correo"**. Con "reducir movimiento" el sello aparece sin animación.

**Cuadro "Tu ruta" (`TuRutaView`):** distancias en línea recta desde la ciudad elegida a las principales ciudades. Solo en pantallas de 1200 px o más.

**Países vecinos:** al pasar el mouse por un vecino se ilumina y muestra "Nuestra red seguirá creciendo · Síguenos en redes", y el enlace "Síguenos en nuestras redes" del pie parpadea más rápido. Al hacer clic, un cometa viaja (0,4 s) y cae el sello "Próximamente".

---

## Pantalla 2: código y contraseña

```
abre el enlace ?solicitud=<id>
  -> tomarSolicitudDeLaUrl(): guarda el id en memoria y lo borra de la barra
  -> ConfirmacionView: seis casillas + contraseña + confirmación
  -> confirmar(): valida en pantalla (6 dígitos, política, coinciden)
  -> RegistroRepository.confirmar(solicitudId, codigo, contrasena)
  -> 201: final "¡Tu cuenta está lista!" + rutas encendidas + onda final del mapa
```

| Parte | Comportamiento |
|---|---|
| Casillas del código | Seis `input` reales. Al escribir pasa a la siguiente; pegar (Ctrl+V) o el autocompletado del celular llenan las seis; borrar en una vacía regresa a la anterior. Solo acepta números. La primera tiene `autocomplete="one-time-code"` |
| Contraseña | Requisitos que se marcan en vivo (`contrasena.rules.ts`, la misma política del user pool), Mostrar/Ocultar y confirmación |
| Tiempos | Cuenta regresiva de 15 min y "Reenviar código" deshabilitado 60 s. Al reenviar, el código anterior deja de servir |
| Errores | `CODIGO_INVALIDO` deja reintentar; `SOLICITUD_VENCIDA`, `SOLICITUD_NO_DISPONIBLE` y `DEMASIADOS_INTENTOS` ofrecen "Empezar de nuevo"; `SOLICITUD_YA_USADA` y `CORREO_YA_REGISTRADO` llevan a iniciar sesión; `POLITICA_CONTRASENA` marca los `Motivos` |
| Final | Reemplaza el formulario (la contraseña escrita sale de la pantalla) y los botones apuntan a `rutaPublica('login')` o `rutaPublica('registro')` |

---

## Celular

- **Disposición:** el mapa ocupa el 38 % superior y la tarjeta va debajo.
- **Barra "Atrás / Continuar":** va pegada al final de la tarjeta (`position: sticky`). Cuando la pregunta es corta queda justo debajo, y cuando es larga se queda abajo al desplazar. Las sugerencias de ciudad y de correo van siempre encima de ella.
- **Campo enfocado:** sube solo por encima de la barra y del teclado (`scroll-margin-bottom`).
- **Toques:** en pantallas táctiles se quitan los efectos de pasar el mouse, que se quedaban pegados al tocar, y el destello azul del navegador.
- **Autocompletado:** los campos conservan el fondo oscuro.
- **Pantallas angostas:** las casillas caben desde 320 px y la barra respeta la muesca del iPhone.
- **Fluidez:** las luces del fondo dejan de moverse en celular para gastar menos batería.

---

## Pruebas unitarias

| Archivo | Qué cubre |
|---|---|
| `la-red/form/ConfirmacionView.spec.ts` | Casillas (escribir, pegar, autocompletar, borrar), validaciones, éxito y errores |
| `la-red/form/avisos.spec.ts` | Avisos flotantes (ver [Piezas comunes](../../../components/la-red-piezas-comunes.md)) |
| `la-red/form/rutas.spec.ts` | Direcciones con y sin `/app` |
| `la-red/form/StepController.spec.ts`, `tuRuta.spec.ts`, `contrasena.rules.spec.ts` | Pasos, distancias y política de contraseña |
| `repository/registro.repository.spec.ts` | Llamadas al API |
| `la-red/scene/Contexto.spec.ts`, `QualityGovernor.spec.ts` | Capas del mapa y regulador de calidad |

---

## Historial de cambios

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-11-03 | Yadilton Torres | Primera versión sobre el mapa "La Red" y zoom a la ciudad elegida. |
| 2026-11-06 | Yadilton Torres | Pantalla del código con seis casillas reales, avisos de lo que va pasando, sello "Paquete enviado", vuelo de la caja más fluido, vecinos con "Síguenos en redes", ajustes para celular y pruebas unitarias. |

---

## Observaciones

- **En PreProd falla `registerContact`:** la ruta no existe en el API de PreProd (M-231). El front ignora ese error y el contacto se crea al confirmar el código.
- **Solicitud temporal:** la del registro se borra sola a las 24 h (`RegistroSolicitudes`). Lo que queda para Comercial está en `RegisterContact`.
- **El enlace del correo apunta al sitio publicado** (`www.logigho.com/app/registro?solicitud=…`). Para probar en local hay que cambiar el principio por `http://localhost:4200/registro?…`.
- **Sin texto que revele cuentas:** la pantalla nunca dice si un correo ya tiene cuenta, porque el API responde igual a propósito.
- **Rol `Red social`:** la cuenta nace con ese rol y sin tiendas. Falta una bienvenida para quien entra solo con ese rol.
