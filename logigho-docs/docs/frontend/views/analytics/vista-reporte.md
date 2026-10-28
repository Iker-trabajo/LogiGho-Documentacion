---
autor: Iker Acevedo Vargas
fecha_creacion: 2026-08-14
ultima_actualizacion: 2026-10-27
estado: desarrollo
nivel: 4
---

# Vista: Visor de Reporte

**Selector:** `app-vista-reporte`<br>
**Ubicación:** `src/app/views/analytics/vista-reporte`<br>
**Ruta:** `/app/analytics/vista-reportes/:id`<br>
**Parámetro:** `id` es el `ReporteId`; no es el `_id` de Mongo.

---

## ¿Qué hace?

Muestra un reporte HTML/BI dentro del sandbox seguro de LogiGho. El rediseño prioriza información entendible: identidad visible, estado, fecha, autor, versión actual, acciones y un lienzo del mismo ancho que la cabecera.

---

## Elementos de la experiencia

| Zona | Qué ofrece |
|---|---|
| Cabecera del reporte | Breadcrumb, nombre, estado, tamaño, fecha, autor y **Reporte ID** persistente con botón de copiar. |
| Controles | Ayuda guiada, selector de versión, Gestionar ETL (si corresponde) y descarga HTML. |
| Vista del informe | Estado de carga, zoom, actualizar, pantalla completa e historial. |
| Historial | Drawer lateral derecho con versión activa, notas, autor, versiones anteriores y descarga vigente. |
| Sandbox | Marco responsivo donde vive el HTML del reporte. |

La cabecera no es fija: acompaña el scroll de la página, evitando cubrir el contenido del reporte. El sandbox usa el mismo ancho máximo visual que la ficha superior para mantener una composición coherente.

---

## Versiones y descarga

- Cambiar la versión solo cambia lo que se visualiza; no modifica cuál es la versión activa publicada.
- **Descargar HTML** entrega el archivo original, no la copia endurecida que se monta dentro del iframe.
- Las notas e historial se abren como drawer, pueden cerrarse con botón, clic en fondo o `Escape`.
- El botón de ayuda ejecuta un tour guiado y abre el drawer automáticamente en los pasos que lo explican.

---

## Flujo de carga

```text
Leer ReporteId de la ruta
  → obtener metadatos y versión solicitada
  → descargar HTML desde S3
  → comprobar SHA-256
  → inyectar CSP de seguridad
  → renderizar dentro del iframe sandbox
  → iniciar telemetría de consumo
```

Actualizar, cambiar versión, descargar y abrir pantalla completa sincronizan la sesión de telemetría para que la auditoría no dependa del intervalo siguiente.

Mientras carga se ve un esqueleto con la forma real de la página y [Pasos de carga](pasos-carga-reporte.md): paso 1 al buscar el reporte, paso 2 al descargar la versión y paso 3 dentro del sandbox. Los clics en la página (`@HostListener('click')`) y dentro del reporte se suman a la telemetría.

---

## Diseño responsive y accesibilidad

- Los controles se reorganizan en filas a medida que se reduce el ancho.
- El drawer usa espacio seguro, márgenes y desplazamiento propio en móvil.
- Los botones tienen texto además de iconos y los cierres responden a `Escape`.
- El ID del reporte se puede copiar sin tener que seleccionar texto manualmente.

### Celular

Estilos en `styles/vista-reporte.scss` (computador) y `styles/vista-reporte.mobile.scss` (celular). En celular solo se adapta el **marco** de la página; el contenido del reporte pertenece al área de datos y no se toca.

| Zona | En celular (≤768 px) |
|---|---|
| Cabecera | Migas en pequeño, título con línea azul, estado y botón "Reporte ID" sin el código largo (sigue copiándolo). |
| Datos | Tamaño, fecha y autor en una sola línea gris. |
| Acciones | Fila que se desliza de lado con textos cortos (Gestionar, Descargar). |
| Vista del informe | Barra delgada de una fila: zoom, ⟳ y ⛶. El historial se abre desde el selector de versión. |
| Reporte | De borde a borde, casi a pantalla completa. |
| Historial | Sube desde abajo como hoja inferior. |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-08-14 | Iker Acevedo Vargas | Visor seguro con descarga, versiones y fullscreen. |
| 2026-09-21 | Iker Acevedo Vargas | Rediseño corporativo responsive, Reporte ID visible, drawer de historial, tour corregido y telemetría integrada. |
| 2026-10-27 | Yadilton Torres | Controles de zoom, esqueleto y pasos de carga, conteo de clics, estilos en `styles/` y marco adaptado a celular. |
