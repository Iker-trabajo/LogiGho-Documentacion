---
autor: Iker Acevedo Vargas
fecha_creacion: 2026-08-14
ultima_actualizacion: 2026-09-21
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
