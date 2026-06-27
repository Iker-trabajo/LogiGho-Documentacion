---

## Autor: Adalberto González
Fecha creacion: 2026-06-27
Estado: desarrollo

# Servicio: ResumenInventarioTourService

**Ubicación:** `src/app/views/logistica/resumen-inventario/resumen-inventario-tour.service.ts`  
**Scope:** `providedIn: 'root'`

---

## ¿Qué hace?

Tiene la lista de pasos que el tour muestra en el módulo de Resumen de Inventario.
Si quieres cambiar lo que dice el tour, agregar un paso o quitarlo, este es el único archivo que tienes que tocar.

---

## Pasos del tour

| # | Elemento que se ilumina       | Título                   | Qué le explica al usuario                                              |
|---|-------------------------------|--------------------------|------------------------------------------------------------------------|
| 1 | Filtro de ecosistema          | Filtro por Ecosistema    | Cómo agrupar tiendas por ecosistema                                    |
| 2 | Filtro de tienda              | Filtro por Tienda        | Cómo filtrar por tienda usando los checkboxes                          |
| 3 | Filtro de bodega              | Filtro por Bodega        | Cómo ver solo los productos de una bodega específica                   |
| 4 | Selector de estado            | Estado del producto      | Cómo alternar entre Activos, Inactivos y Todos                         |
| 5 | Grid de KPIs                  | Indicadores clave (KPIs) | Qué muestra cada tarjeta y que las clickeables abren el detalle        |
| 6 | Botón Resumen                 | Botón Resumen            | Que abre el resumen consolidado y que también funciona con Alt + R     |
| 7 | Botón del tour y atajos       | Atajos de teclado        | Los atajos Alt + R, Alt + B y Alt + P                                  |

---

## Historial de cambios

| Fecha      | Autor              | Cambio       |
| ---------- | ------------------ | ------------ |
| 2026-06-27 | Adalberto González | Creación.    |

---

## Observaciones

- Los `id` de los elementos están definidos en el template de `ResumenInventarioComponent`. Si alguno cambia de nombre, ese paso simplemente no aparece en el tour.
- Para agregar un paso nuevo: agrega un objeto al array `steps` y pon el `id` correspondiente en el HTML del módulo.
