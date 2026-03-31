---
autor: 
fecha_creacion: YYYY-MM-DD
ultima_actualizacion: YYYY-MM-DD
estado: desarrollo | produccion | deprecado
nivel: 1 | 2 | 3
---

# Vista: NombreVista

**Autor:** Nombre Apellido  
**Selector:** `app-nombre-vista`  
**Ubicación:** `SitioLogiGho/src/app/views/nombre-vista`

---

## ¿Qué hace?

Descripción breve en 2-3 líneas. Qué muestra esta vista, para qué sirve, qué problema resuelve al usuario.

---

## Ruta

| Propiedad | Valor |
|---|---|
| **Ruta** | `/app/nombre-vista` |
| **Título de página** | Nombre que aparece en el navegador |
| **Guard** | `AuthGuard` | `RolGuard` | Ninguno |
| **Rol requerido** | `admin` | `vendedor` | Todos |
| **Parámetros de URL** | `:id`, `:slug` — o ninguno |

### Definición en `app.routes.ts`

```typescript
{
  path: 'nombre-vista',
  component: NombreVistaComponent,
  canActivate: [AuthGuard],
  title: 'Título de la página'
}
```

---

## Parámetros de URL

> Solo si la ruta recibe parámetros. Ej: `/app/tarjetas/:id`

| Parámetro | Tipo | Descripción |
|---|---|---|
| `id` | `string` | ID del recurso a mostrar |

---

## Estructura de archivos

```
nombre-vista/
├── nombre-vista.component.ts
├── nombre-vista.component.html
├── nombre-vista.component.scss
└── components/               (subcomponentes locales si tiene)
    └── sub-componente/
```

---

## Secciones de la vista

| # | Sección | Descripción |
|---|---|---|
| 1 | **Encabezado** | Título, breadcrumbs, acciones principales |
| 2 | **Contenido principal** | Tabla, formulario, detalle, etc. |
| 3 | **Acciones** | Botones de guardar, cancelar, eliminar |

---

## Propiedades del componente

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `dato` | `NombreTipo` | `null` | Dato principal que muestra la vista |
| `cargando` | `boolean` | `false` | Controla el estado de loading |
| `error` | `string` | `''` | Mensaje de error si falla la carga |

---

## Flujo de inicialización

```
ngOnInit()
  -> Obtiene parámetros de URL (si aplica)
  -> cargarDatos()
     -> NombreService.obtener(id)
        -> API GET /ruta
  -> Renderiza la vista
```

---

## Métodos

### `cargarDatos()`
**Descripción:** Carga los datos necesarios para renderizar la vista.

**Proceso:**
1. Activa estado de carga (`cargando = true`)
2. Llama al servicio correspondiente
3. Asigna los datos a las propiedades del componente
4. Desactiva estado de carga

**Manejo de error:** Si falla, asigna mensaje a `error` y muestra feedback al usuario.

---

## Servicios utilizados

| Servicio | Métodos usados | Propósito |
|---|---|---|
| `NombreService` | `obtener()`, `guardar()` | CRUD del recurso |
| `Router` | `navigate()` | Redirecciones |

---

## Endpoints que consume

| Método | Ruta | Cuándo |
|---|---|---|
| `GET` | `/api/v1/recurso` | Al cargar la vista |
| `POST` | `/api/v1/recurso` | Al guardar el formulario |

---

## Estados de la vista

| Estado | Descripción | Qué muestra |
|---|---|---|
| Cargando | Esperando respuesta del API | Spinner / skeleton |
| Con datos | Datos cargados correctamente | Contenido normal |
| Error | Falló la carga | Mensaje de error + botón reintentar |
| Vacío | No hay datos | Mensaje de estado vacío |

---

## Navegación desde esta vista

| Acción del usuario | Navega a |
|---|---|
| Click en "Ver detalle" | `/app/recurso/:id` |
| Click en "Volver" | `/app/lista` |

---

## Subcomponentes

| Componente | Selector | Descripción |
|---|---|---|
| `NombreComponent` | `app-nombre` | Para qué sirve |

---

## Estilos

> Solo documentar si tiene estilos específicos importantes o variables SCSS propias.

---

## Changelog de la vista

| Fecha | Autor | Cambio |
|---|---|---|
| YYYY-MM-DD | Nombre | Descripción |

---

## Observaciones

> Deuda técnica, comportamientos especiales, cosas no obvias.

- Observación 1
