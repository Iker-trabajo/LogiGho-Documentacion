---
autor: Iker Acevedo Vargas
fecha_creacion: 2026-08-14
ultima_actualizacion: 2026-10-27
estado: desarrollo
nivel: 5
---

# Componente: Sandbox de Reporte

**Selector:** `app-reporte-sandbox`

**Ubicación:** `src/app/views/analytics/components/reporte-sandbox`

**Acceso:** Interno — no es una ruta, lo montan `GestionReportesComponent` y `VistaReporteComponent`

---

## ¿Qué hace?

Renderiza HTML no confiable de forma aislada. Es el componente donde vive la Capa 1 (iframe sandbox) y la Capa 5 (pantalla completa sin pestaña nueva) del [modelo de seguridad del módulo](overview.md#seguridad-del-visor). Se reutiliza en previsualización y en el visor de usuario.

---

## Decoradores

| Decorador | Nombre | Tipo | Descripción |
| --- | --- | --- | --- |
| `@Input` | `htmlEndurecido` | `string` (required) | HTML **ya** procesado por `ReporteSanitizerService.endurecerHtml()`. Este componente nunca llama a `endurecerHtml()` por su cuenta — el contrato es que llega listo. |
| `@Input` | `mostrarBotonPropio` | `boolean` (default `true`) | `false` cuando el contenedor ya trae su propio control de pantalla completa (evita duplicarlo — así lo usa `VistaReporteComponent`) |

---

## Propiedades clave

| Propiedad | Tipo | Descripción |
| --- | --- | --- |
| `urlSegura` | `SafeResourceUrl \| null` | Blob URL del HTML, envuelto con `DomSanitizer.bypassSecurityTrustResourceUrl()` |
| `enPantallaCompleta` | `boolean` | Sincronizado con el evento nativo `fullscreenchange` |
| `cargando` / `error` | `boolean` / `string \| null` | Estado de render del blob. `cargando` sigue en `true` hasta el `(load)` del iframe (máximo 10 s, `MILISEGUNDOS_RESPALDO_CARGA`). |
| `zoom` / `zoomAjustado` | `number` | Escala visual del iframe (1 = 100 %) y la escala que hace caber el reporte en celular. |

| `@Output` | Descripción |
| --- | --- |
| `clicEnReporte` | Se emite con cada clic dentro del reporte. No trae ningún dato del contenido. |

---

## Flujo principal

```
ngOnChanges(htmlEndurecido)
  -> renderizar()
     -> new Blob([htmlEndurecido], {type: 'text/html'})
     -> URL.createObjectURL(blob)
     -> sanitizer.bypassSecurityTrustResourceUrl(blobUrl)
     -> urlSegura = ...
ngOnDestroy()
  -> URL.revokeObjectURL(blobUrlActual)   // libera memoria, no deja blobs huérfanos
```

---

## Métodos clave

### `alternarPantallaCompleta()`

Fullscreen API nativa (`Element.requestFullscreen()` / `document.exitFullscreen()`) sobre el `<div>` contenedor — nunca sobre el `<iframe>` directamente, para poder mostrar el botón de salir superpuesto durante el fullscreen. Deliberadamente **no** usa `window.open()`: abrir el `blob:` como documento de primer nivel comprometería el aislamiento descrito en el [modelo de seguridad](overview.md#seguridad-del-visor).

### Zoom

| Método | Qué hace |
| --- | --- |
| `acercar()` / `alejar()` | Recorre los niveles de `50 %` a `200 %` (`NIVELES_ZOOM`). |
| `restablecerZoom()` | Vuelve a 100 % en computador o a **Ajustado** en celular. |
| `ajustarAlAncho()` | Lo llama un `ResizeObserver`. Si el contenedor mide menos de 768 px, dibuja el reporte a **1024 px** de ancho (`ANCHO_DISENO_REPORTE`) y lo encoge para que quepa completo. |

El zoom solo escala el `<iframe>` con `transform: scale()`; el contenido del reporte no se toca. Los botones viven fuera del componente (visor), y en pantalla completa aparece una barra de zoom flotante dentro del contenedor.

### Aviso de clics dentro del reporte

El iframe está aislado, así que la página no puede escuchar sus clics. `endurecerHtml()` inyecta un script pequeño (`SCRIPT_AVISO_CLICS`, en `reporte-sanitizer`) que avisa con `postMessage` un mensaje `{ tipo: 'logigho-reporte-clic' }`.

El componente solo acepta el aviso si `evento.source` es la ventana de **su propio** iframe y el mensaje tiene exactamente esa forma. El origen del iframe es opaco (`null`), por eso no se filtra por `event.origin`.

### Carga

Mientras carga se muestra `app-pasos-carga-reporte` en el paso 3 ("Preparando la vista"), ver [Pasos de carga](pasos-carga-reporte.md). El iframe aparece con una transición suave de opacidad. El `load` inicial de `about:blank` se descarta: solo cuenta el del blob del reporte.

---

## Interfaz HTML del `<iframe>`

```html
<iframe
  [src]="urlSegura"
  sandbox="allow-scripts allow-downloads allow-modals"
  referrerpolicy="no-referrer"
  ...
></iframe>
```

`sandbox` es **texto estático**, nunca `[sandbox]="..."`. Angular arroja `NG0910` en runtime si el atributo `sandbox` de un `<iframe>` con `[src]` dinámico está bindeado — es una protección deliberada del framework: con un binding, el valor podría variar y colar `allow-same-origin` en tiempo de ejecución, justo lo que este control impide. Si hay que cambiar los permisos concedidos, se edita el atributo directamente en la plantilla.

Tokens concedidos y por qué:

| Token | Por qué se concede |
| --- | --- |
| `allow-scripts` | El reporte necesita ejecutar su JavaScript |
| `allow-downloads` | Los reportes de ejemplo tienen KPIs con descarga de datos al clic |
| `allow-modals` | Para `alert()`/`confirm()` dentro del reporte |

Tokens **nunca** concedidos: `allow-same-origin` (el que anularía todo lo demás), `allow-top-navigation*`, `allow-forms`, `allow-popups*`, `allow-pointer-lock`, `allow-presentation`.

---

## Observaciones

- `:host { display: block; width: 100%; height: 100%; }` es obligatorio en el SCSS — sin él, el componente se comporta como elemento inline dentro de cualquier contenedor flex padre y se encoge al tamaño de su contenido en vez de llenar el espacio disponible (bug real detectado: el iframe caía al tamaño intrínseco por defecto del navegador, ~300×150px).
- Cualquier contenedor flex que use este componente debe darle `flex: 1; min-width: 0; min-height: 0;` explícitamente — un flex item no crece por defecto sin `flex-grow`.

---

## Changelog

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-08-14 | Iker Acevedo Vargas | Versión inicial con sandbox de 3 tokens, blob URL, y Fullscreen API nativa reemplazando el botón de pestaña nueva |
| 2026-09-21 | Iker Acevedo Vargas | Integración con el visor rediseñado, cuyos controles externos evitan duplicar el botón interno de pantalla completa. |
| 2026-10-27 | Yadilton Torres | Zoom con modo Ajustado para celular, aviso de clics por `postMessage`, carga hasta el `load` del iframe con transición y, en celular, reporte de borde a borde sin arrastrar la página. |
