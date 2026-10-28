---
autor: Iker Acevedo Vargas
fecha_creacion: 2026-08-14
ultima_actualizacion: 2026-10-28
estado: desarrollo
nivel: 4
---

# Vista: Gestión de Reportes

**Selector:** `app-gestion-reportes`<br>
**Ubicación:** `src/app/views/analytics/gestion-reportes`<br>
**Ruta:** `/app/analytics/gestion-reportes`

---

## ¿Qué hace?

Es el espacio del área de datos para publicar reportes HTML, subir nuevas versiones, editar información del reporte, administrar roles con acceso y archivar o reactivar contenido. La interfaz está preparada para nombres largos, tablas y modales en pantallas pequeñas.

---

## Acciones disponibles

| Acción | Resultado |
|---|---|
| Nuevo reporte | Valida el archivo y abre los metadatos antes de publicar. |
| Subir nueva versión | Agrega una versión al mismo reporte y mueve la versión actual. |
| Editar metadatos | Actualiza nombre, descripción, área/categoría, estado y roles **sin crear versión**. |
| Editar roles | Agrega o quita roles existentes desde la colección `Roles`. |
| Historial | Consulta versiones del reporte o actividad global. |
| Archivar / reactivar | Cambia disponibilidad sin borrar archivos ni historial. |
| Áreas organizacionales | Crea y edita áreas reutilizadas por reportes y telemetría. |

---

## Roles, áreas y versiones

- Los botones de rol son multiselección real: al editar, muestran roles guardados y permiten marcar o desmarcar otros.
- Los roles se consultan desde la colección `Roles` mediante el método genérico; no existe una lista escrita en el componente.
- Las categorías disponibles provienen de áreas organizacionales activas. Se crea una nueva sin cambiar código.
- Una edición de metadatos usa `soloMetadatos`; no genera archivo, no toca S3 y no crea una versión nueva.

---

## Menú de acciones y accesibilidad

El menú de tres puntos se ubica con coordenadas de ventana y se recalcula al hacer scroll, redimensionar o cambiar zoom. Así no se desplaza ni queda recortado por el contenedor de tabla.

`Escape` cierra el elemento abierto más reciente: menú, previsualización, historial, reglas o modal. Los formularios validan campos antes de enviar y bloquean doble envío mientras guardan.

---

## Flujo de publicación

```text
Seleccionar o soltar HTML
  → validar contenido y reglas de seguridad
  → completar metadatos y roles
  → comprimir, calcular hash y cargar archivo
  → guardar versión y metadatos
  → confirmar resultado y recargar la tabla
```

La fecha de última actualización se presenta, por ejemplo, como `21 de ago de 2026`.

---

## Pantallas de carga

- Mientras carga la tabla se muestran filas fantasma con la misma forma de la tabla (en celular, con forma de tarjeta).
- La vista previa muestra [Pasos de carga](pasos-carga-reporte.md) ocupando todo el modal mientras se descarga la versión.

---

## Vista previa

```text
[👁]  VISTA PREVIA
      Seguimiento Efectividad Hora · Operaciones   [− Ajustado +] [⛶] [ ✕ ]
┌──────────────────────────────────────────────────────────────────────┐
│                  (reporte, de borde a borde)                         │
```

- Ocupa el 95 % del ancho (hasta 1.400 px) y el 92 % del alto, por encima del encabezado de la plataforma (`z-index: 3000`).
- Un solo marco: el reporte va de borde a borde bajo la barra de título; el contenido del reporte no se modifica.
- **Zoom y pantalla completa** van en la barra de título, fuera del reporte, para no tapar sus botones. Usan los métodos del sandbox (`acercar()`, `alejar()`, `restablecerZoom()`, `alternarPantallaCompleta()`) a través de `@ViewChild(ReporteSandboxComponent) sandboxPreview`; por eso el sandbox se monta con `[mostrarBotonPropio]="false"`.
- En celular se abre a pantalla completa con una barra de título delgada; la etiqueta "Vista previa" se oculta y los botones miden 40 px. Desde **Ajustado**, `+` agranda el reporte y se recorre con el dedo.

Todos los modales de la vista tienen un botón **✕** arriba a la derecha (historial de un reporte, historial de actividad, publicar/editar metadatos, reglas, áreas y vista previa). En el de metadatos, la ✕ se desactiva mientras se publica, igual que Cancelar.

---

## Versión para celular

Estilos en `styles/gestion-reportes.scss` (computador) y `styles/gestion-reportes.mobile.scss` (celular). Los modales hijos (`modal-metadatos-reporte`, `reglas-autores-reporte`, `modal-area-organizacional`) tienen su propio `*.component.mobile.scss`, porque los estilos de la página no llegan a sus componentes.

| Zona | En celular (≤768 px) |
|---|---|
| Página | Ocupa todo el ancho. |
| Altura | Sube 36 px para compensar el margen del contenedor (20 px) y el que deja el encabezado (`mb-4` menos el `-2%` del cuerpo); el título queda a 8 px de la barra. |
| Título | 22 px con línea azul de acento; el subtítulo pierde la segunda frase. |
| Acciones | Fila de íconos **Reglas**, **Historial**, **Áreas** y **?**; debajo **+ Nuevo Reporte** a lo ancho. |
| Zona de arrastre | Se oculta: arrastrar archivos no existe en táctil y "+ Nuevo Reporte" hace lo mismo. |
| Tabla | Cada fila es una tarjeta compacta (ver abajo) con el menú ⋮ en la esquina. |
| Menú ⋮ y modales | Suben desde abajo como hoja inferior, por encima del encabezado. |

Tamaños de pantalla usados: 860, 768 y 400 px.

### Tarjeta de reporte en celular

```text
[Activo] [Operaciones]                      ⋮
Seguimiento Efectividad Hora
🕘 28 de oct de 2026
👤 haroldpc258@gmail.com
───────────────────────────────────────────
Visible para
[CEO] [BPO] [COO] [Datos] [+5 más]
```

- Las celdas se acomodan con `grid-template-areas`; las etiquetas en mayúsculas de cada celda se ocultan, salvo "Visible para".
- Fecha y autor llevan su ícono (`.solo-movil`, oculto en computador).
- Los roles se ven como chips: los primeros `ROLES_VISIBLES` (4) y un chip **"+N más"** que despliega todos y cambia a "Ver menos". El estado por reporte se guarda en `rolesExpandidos` y lo alterna `alternarRoles()`.
- En computador la columna sigue mostrando los roles separados por comas.

---

## Servicios relacionados

| Servicio | Responsabilidad |
|---|---|
| `ReportesAnalyticsService` | Lista, publica, versiona, edita metadatos y cambia estado. |
| `ReporteSanitizerService` | Valida y endurece HTML antes de previsualizarlo. |
| `AreasOrganizacionalesService` | Carga áreas activas para categorías. |
| `ReporteSandboxComponent` | Previsualiza sin exponer la sesión de plataforma. |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-08-14 | Iker Acevedo Vargas | Publicación, versionado, historial y validación de archivos. |
| 2026-09-21 | Iker Acevedo Vargas | Edición real de roles sin versionar, áreas dinámicas, menú estable en zoom/scroll, validaciones y mejoras responsive. |
| 2026-10-27 | Yadilton Torres | Esqueletos y pasos de carga, vista previa grande con un solo marco, botón ✕ en todos los modales, estilos en `styles/` y versión para celular (fila de íconos, tarjetas y hojas inferiores). |
| 2026-10-28 | Yadilton Torres | Zoom y pantalla completa en la barra de la vista previa; tarjeta compacta en celular con roles como chips ("+N más"); título más arriba en celular. |
