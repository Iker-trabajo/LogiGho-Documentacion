---
autor: Iker Acevedo Vargas
fecha_creacion: 2026-08-14
ultima_actualizacion: 2026-09-21
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

---

## Diseño responsive y accesibilidad

- Los controles se reorganizan en filas a medida que se reduce el ancho.
- El drawer usa espacio seguro, márgenes y desplazamiento propio en móvil.
- Los botones tienen texto además de iconos y los cierres responden a `Escape`.
- El ID del reporte se puede copiar sin tener que seleccionar texto manualmente.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-08-14 | Iker Acevedo Vargas | Visor seguro con descarga, versiones y fullscreen. |
| 2026-09-21 | Iker Acevedo Vargas | Rediseño corporativo responsive, Reporte ID visible, drawer de historial, tour corregido y telemetría integrada. |
