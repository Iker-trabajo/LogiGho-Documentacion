---
autor: Yadilton Torres
fecha_creacion: 2026-10-27
ultima_actualizacion: 2026-10-27
estado: desarrollo
nivel: 3
---

# Componente: Pasos de carga del reporte

**Selector:** `app-pasos-carga-reporte`<br>
**Ubicación:** `src/app/views/analytics/components/pasos-carga-reporte`<br>
**Acceso:** Interno — no es una ruta, lo usan `VistaReporteComponent`, `GestionReportesComponent` y `ReporteSandboxComponent`.

---

## ¿Qué hace?

Muestra en qué etapa va la apertura de un reporte, en lugar de un spinner suelto distinto en cada fase. El usuario ve que la carga avanza y en qué punto está.

```text
● Buscando el reporte
● Descargando la versión
○ Preparando la vista
```

---

## Decoradores

| Decorador | Nombre | Tipo | Descripción |
| --- | --- | --- | --- |
| `@Input` | `paso` | `PasoCargaReporte` (`1 \| 2 \| 3`, required) | Etapa en curso. Las anteriores se marcan como hechas. |

---

## Pasos

| Paso | Texto | Quién lo muestra |
| --- | --- | --- |
| 1 | Buscando el reporte | Visor, mientras obtiene los metadatos. |
| 2 | Descargando la versión | Visor y vista previa de Gestión, mientras baja el HTML desde S3. |
| 3 | Preparando la vista | Sandbox, hasta que el iframe termina de cargar (máximo 10 s). |

---

## Historial de cambios

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-10-27 | Yadilton Torres | Componente inicial con los tres pasos de carga. |
