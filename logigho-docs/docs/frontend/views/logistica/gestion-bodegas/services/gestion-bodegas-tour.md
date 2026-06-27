---

## Autor: Adalberto González
Fecha creacion: 2026-06-27
Estado: desarrollo

# Servicio: GestionBodegasTourService

**Ubicación:** `src/app/views/logistica/gestion-bodegas/gestion-bodegas-tour.service.ts`  
**Scope:** `providedIn: 'root'`

---

## ¿Qué hace?

Tiene la lista de pasos que el tour muestra en el módulo de Gestión de Bodegas.
Si quieres cambiar lo que dice el tour, agregar un paso o quitarlo, este es el único archivo que tienes que tocar.

---

## Pasos del tour

| # | Elemento que se ilumina     | Título                 | Qué le explica al usuario                                                  |
|---|-----------------------------|------------------------|----------------------------------------------------------------------------|
| 1 | Grid de KPIs                | Indicadores de Bodegas | Qué muestra cada tarjeta: total, stock y bodegas bajo el mínimo            |
| 2 | Toolbar de filtros          | Búsqueda y filtros     | Cómo buscar por nombre, filtrar por ciudad y cambiar el estado             |
| 3 | Tabla de bodegas            | Listado de bodegas     | Qué se puede hacer desde cada fila: editar, activar/desactivar, gestionar  |
| 4 | Botón Asignar producto      | Asignar producto       | Cómo vincular un producto y una tienda a una bodega *(solo si el rol lo permite)* |
| 5 | Botón Nueva bodega          | Nueva bodega           | Cómo crear una bodega nueva                                                |
| 6 | Botón del tour              | Tour guiado            | Que este botón relanza el tour y los atajos de teclado disponibles         |

---

## Historial de cambios

| Fecha      | Autor              | Cambio       |
| ---------- | ------------------ | ------------ |
| 2026-06-27 | Adalberto González | Creación.    |

---

## Observaciones

- Los `id` de los elementos están definidos en el template de `GestionBodegasComponent`. Si alguno cambia de nombre, ese paso simplemente no aparece en el tour.
- El paso 4 (Asignar producto) solo se muestra si el usuario tiene el rol que habilita ese botón. No hace falta hacer nada especial para eso, el componente del tour lo maneja solo.
- Para agregar un paso nuevo: agrega un objeto al array `steps` y pon el `id` correspondiente en el HTML del módulo.
