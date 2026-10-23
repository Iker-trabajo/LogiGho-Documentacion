---

## Autor: Yadilton Torres

Fecha creacion: 2026-10-21
Estado: desarrollo
Tipo: componente

# Componente: CampanaNotificacionesComponent

**Selector:** `app-campana-notificaciones`
**Ubicación:** `src/app/layout/default-layout/default-header/components/campana-notificaciones/campana-notificaciones.component.ts`
**Acceso:** Autenticado (vive en el header de la plataforma)

---

## ¿Qué hace?

Es la campana del header. Muestra cuántas notificaciones nuevas tiene el usuario y, al tocarla, abre un panel con las pendientes, el historial y el detalle de cada una. Desde el detalle el equipo responde una solicitud o el usuario marca una respuesta como "Entendido".

Reemplaza la campana anterior del header, que mostraba un "1" fijo y no hacía nada. Los datos y la lógica están en [NotificacionesService](../core/notificaciones-service.md).

---

## Partes

| Parte | Qué muestra |
| --- | --- |
| **Botón campana** | Icono `cilBell` con contador rojo de nuevas (`9+` si son más de 9) |
| **Cabecera del panel** | Degradado azul de la página con círculos de adorno. Título "Notificaciones", chip "N nuevas" con un punto amarillo que late ("Sin mensajes nuevos" / "No hay notificaciones"), botón **Actualizar** (gira mientras consulta) y **X** para cerrar |
| **Pestañas** | Tipo segmentado (la activa en blanco): **Pendientes** (con cuántas hay) e **Historial** (últimas 20 atendidas) |
| **Lista** | Cada notificación es una tarjeta blanca sobre fondo claro, con icono en degradado naranja o verde según su tipo, quién la envió y hace cuánto ("Ahora", "Hace 5 min", "Ayer"…). En el historial, quién respondió. Pie con "Actualizado hace…" |
| **Vacío** | "No tienes notificaciones" o "Aún no hay historial" |
| **Detalle** | Botón Volver y X para cerrar, quién la envió y cuándo, el mensaje, los datos del contexto (Colección, Formato, Fechas, Registros, Columnas, Límite permitido) |
| **Acciones del detalle** | Tipo `responder`: cuadro "Respuesta para <correo>" (máx. 500 caracteres) y botón **Responder y marcar como hecha** (activo solo con texto). Tipo `entendido`: botón **Entendido** |
| **Ya atendida** | Bloque verde "Respondió <correo>", la fecha y la respuesta, sin botones |

---

## Flujo

```
ngOnInit()
  -> servicio.iniciar()                       // revisión periódica

alternar()                                    // tocar la campana
  -> abre en la pestaña Pendientes y consulta de nuevo (para no mostrar datos de hace 2 min)

verPestana('historial')
  -> servicio.revisarHistorial()             // el historial se consulta solo al abrirlo

abrir(notificacion)
  -> muestra el detalle
  -> si era NUEVA → servicio.marcarLeida()

marcarHecha()
  -> servicio.marcarHecha(notificacion, respuesta)
  -> si tiene respuesta → aviso "Respuesta enviada" (le llegó a <correo>, pasó al historial)
  -> RespuestaNoEnviadaError     → aviso "Quedó como hecha" (no se pudo avisar, escríbele por otro medio)
  -> EscrituraDesactivadaError   → aviso "Por ahora los cambios no se guardan"
  -> otro error                  → aviso "No se pudo guardar"

ngOnDestroy()
  -> servicio.detener()
```

---

## Teclado y cierre

| Acción | Resultado |
| --- | --- |
| `Escape` en el detalle | Vuelve a la lista |
| `Escape` en la lista | Cierra el panel |
| Clic fuera del panel | Cierra el panel. Se usa `composedPath()` y no `contains`, porque al pasar de la lista al detalle el botón tocado ya salió del DOM |

---

## Celular (hasta 576px)

El panel queda fijo y a todo el ancho de la pantalla, y la X para cerrar es de 40px para tocarla fácil. Con `prefers-reduced-motion` se quitan las animaciones.

---

## Tour

El botón de la campana tiene `id="tour-campana-notificaciones"`, así otras pantallas pueden mostrarla en su tour. Hoy lo usa el tour de [Exportación de datos](../views/gestion-datos/export-data/export-data.md) para explicar que ahí llega la respuesta del equipo de desarrollo.

---

## Uso en el header

En `default-header.component.html` el enlace viejo `<a class="header-action notification">` se cambió por:

```html
<app-campana-notificaciones></app-campana-notificaciones>
```

y en `default-header.component.scss` se borró el CSS del "1" falso y su animación.

---

## Pruebas

`campana-notificaciones.component.spec.ts`: 20 pruebas (contador, abrir y cerrar, pestañas, detalle, responder, entendido, errores, Escape y clic afuera).

---

## Historial de cambios

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-10-21 | Yadilton Torres | Creación de la campana con contador, panel de pendientes y detalle; reemplaza la campana fija del header |
| 2026-10-21 | Yadilton Torres | Respuesta obligatoria al atender una solicitud, botón "Entendido" y pestaña Historial |
| 2026-10-21 | Yadilton Torres | Diseño final: cabecera en degradado azul, pestañas segmentadas, notificaciones en tarjetas, X para cerrar e `id` para los tours |

---

## Observaciones

- **Los iconos de cada tipo** (`cilCloudDownload`, `cilTask`, `cilBell`) deben estar registrados en el `IconSetService` del layout.
- **No probado todavía en la app real**, solo con pruebas y vista previa en PC y celular.
