---
autor: Iker Acevedo Vargas
fecha_creacion: 2026-08-14
ultima_actualizacion: 2026-08-14
estado: desarrollo
nivel: 5
---

# Componente: Sandbox de Reporte

**Selector:** `app-reporte-sandbox`

**Ubicación:** `src/app/views/analytics/components/reporte-sandbox`

**Acceso:** Interno — no es una ruta, lo montan `GestionReportesComponent` y `VistaReporteComponent`

---

## ¿Qué hace?

Renderiza HTML no confiable de forma aislada. Es el componente donde vive la Capa 1 (iframe sandbox) y la Capa 5 (pantalla completa sin pestaña nueva) del [modelo de seguridad del módulo](overview.md#modelo-de-seguridad). Reutilizado tal cual en tres lugares: previsualización en Gestión de Reportes, visor de usuario, y (potencialmente) cualquier futura vista que necesite renderizar HTML no confiable.

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
| `cargando` / `error` | `boolean` / `string \| null` | Estado de render del blob |

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

Fullscreen API nativa (`Element.requestFullscreen()` / `document.exitFullscreen()`) sobre el `<div>` contenedor — nunca sobre el `<iframe>` directamente, para poder mostrar el botón de salir superpuesto durante el fullscreen. Deliberadamente **no** usa `window.open()`: ver la decisión descartada en el [overview](overview.md#decisión-descartada-abrir-en-pestaña-nueva).

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
