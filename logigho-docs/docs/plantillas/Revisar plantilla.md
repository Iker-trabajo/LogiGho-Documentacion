---
autor: 
fecha_creacion: YYYY-MM-DD
ultima_actualizacion: YYYY-MM-DD
estado: desarrollo | produccion | deprecado
nivel: 1 | 2 | 3
---
 
# Vista: NombreVista
 
**Selector:** `app-nombre-vista`  
**Ubicación:** `src/app/views/nombre-vista`
 
---
 
## ¿Qué hace?
 
2-3 líneas. Qué muestra, para quién, qué problema resuelve.
 
---
 
## Propiedades principales
 
| Propiedad | Tipo | Descripción |
|---|---|---|
| `dato` | `Tipo` | Para qué sirve |
 
> Solo las que no son obvias por su nombre. Si tiene más de 10, documentar solo las críticas.
 
---
 
## Servicios y endpoints
 
| Servicio | Método | Endpoint | Cuándo |
|---|---|---|---|
| `NombreService` | `obtener()` | `GET /api/recurso` | Al inicializar |
 
---
 
## Observaciones
 
> Deuda técnica, comportamientos no obvios, decisiones de diseño importantes.
 
- Observación 1
 
---
 
<!-- NIVEL 2 en adelante: agregar estas secciones -->
 
## Secciones de la vista
 
| # | Sección | Descripción |
|---|---|---|
| 1 | **Filtros** | ... |
 
---
 
## Flujo de inicialización
 
```
ngOnInit()
  -> servicio.obtener()
  -> procesar resultado
  -> renderizar
```
 
---
 
## Métodos clave
 
### `nombreMetodo()`
Qué hace en 1-2 líneas. Solo si tiene lógica no obvia.
 
---
 
## Estados de la vista
 
| Estado | Qué muestra |
|---|---|
| Cargando | Spinner |
| Con datos | Tabla / contenido |
| Error | Mensaje + reintentar |
| Vacío | Estado vacío |
 
---
 
<!-- NIVEL 3 en adelante: agregar estas secciones -->
 
## Interfaces
 
### `NombreTipo`
Solo si no está documentada en otro lado.
 
| Campo | Tipo | Descripción |
|---|---|---|
| `campo` | `string` | ... |
 
---
 
## Subcomponentes
 
| Componente | Selector | Qué hace |
|---|---|---|
| `NombreComponent` | `app-nombre` | ... |
 
---
 
## Changelog
 
| Fecha | Autor | Cambio |
|---|---|---|
| YYYY-MM-DD | Nombre | ... |