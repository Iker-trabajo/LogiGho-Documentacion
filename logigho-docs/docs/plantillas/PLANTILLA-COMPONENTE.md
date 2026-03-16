---
autor: 
fecha_creacion: YYYY-MM-DD
ultima_actualizacion: YYYY-MM-DD
estado: desarrollo | produccion | deprecado
nivel: 1 | 2 | 3
---

# Nombre del Componente

**Autor:** Nombre Apellido  
**Selector:** `app-nombre-componente`  
**Ubicación:** `SitioLogiGho/src/app/[ruta]/nombre-componente`

---

## ¿Qué hace?

Descripción breve en 2-3 líneas. Qué problema resuelve, qué muestra, para qué sirve.

---

## Roles y acceso

| Acceso | Descripción |
|---|---|
| Público | Cualquier visitante puede acceder |
| Autenticado | Requiere login |
| Rol específico | Requiere rol: `admin`, `vendedor`, etc. |

---

## Estructura de archivos

```
nombre-componente/
├── nombre-componente.component.ts
├── nombre-componente.component.html
├── nombre-componente.component.scss
└── components/               (si tiene subcomponentes)
    └── sub-componente/
```

---

## Secciones de la página

> Solo para componentes tipo vista/página. Eliminar si no aplica.

1. **Sección 1** — Descripción breve
2. **Sección 2** — Descripción breve

---

## Propiedades del componente

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `propiedad` | `string` | `''` | Para qué sirve |

---

## Métodos

### `nombreMetodo(param: tipo): retorno`

**Descripción:** Qué hace este método.

**Parámetros:**
- `param` — Descripción del parámetro

**Proceso:**
1. Paso 1
2. Paso 2
3. Paso 3

**Retorna:** Descripción de lo que retorna

---

## Flujo principal

```
ngOnInit() / ngAfterViewInit()
  -> metodo1()
  -> metodo2()
     -> metodo3()
```

---

## Dependencias externas

| Librería | Versión | Uso |
|---|---|---|
| Chart.js | 4.x | Gráficos interactivos |
| — | — | — |

## Servicios Angular utilizados

| Servicio | Métodos usados | Propósito |
|---|---|---|
| `Router` | `navigate()` | Navegación entre rutas |
| — | — | — |

---

## Estilos

### Variables SCSS principales

| Variable | Valor | Uso |
|---|---|---|
| `$color-primario` | `#0e127f` | Color principal |

### Animaciones

| Animación | Duración | Uso |
|---|---|---|
| `fadeIn` | 1s | Entrada de elementos |

---

## Subcomponentes

| Componente | Selector | Descripción |
|---|---|---|
| `NombreComponent` | `app-nombre` | Para qué sirve |

---

## Changelog del componente

| Fecha | Autor | Cambio |
|---|---|---|
| YYYY-MM-DD | Nombre | Descripción del cambio |

---

## Observaciones

> Deuda técnica, malas prácticas conocidas, cosas pendientes de mejorar.

- Observación 1
- Observación 2
