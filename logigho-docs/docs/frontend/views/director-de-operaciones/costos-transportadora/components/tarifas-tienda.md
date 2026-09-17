## Autor: Iker Acevedo
Fecha creación: 2026-09-16
Estado: preprod
Tipo: componente

# Tarifas por Tienda

**Selector:** `app-tarifas-tienda`
**Ubicación:** `src/app/views/director-de-operaciones/costos-transportadora/components/tarifas-tienda`

---

## ¿Qué hace?

Es la pantalla donde se configura lo que realmente cuesta liquidar una guía: comisión de recaudo, seguro, flete por trayecto (con su tabla de incrementos por kilo si la política lo requiere), y el mínimo fijo de servicio. Una tienda sin tarifa propia usa la **Genérica** del mismo tipo (Entrega o Devolución) — es la pantalla más grande y con más reglas de las 5 pestañas.

---

## El "Alcance" ya no es un campo que el usuario piense aparte

En una versión anterior, "¿es de una tienda puntual o Genérico?" era un campo separado del selector de tienda — dos decisiones que en la práctica siempre iban de la mano. Ahora es una sola: "Genérico" es una opción más dentro del mismo selector de Tienda, igual que cualquier tienda real. Elegir "Genérico" pone `alcance = Generico` e `idTienda = null` automáticamente.

---

## Filtros por tienda/ecosistema — el bug que corrigió el diseño con signals

Los filtros de la tabla principal son **signals**, no propiedades planas, a propósito. `tarifasFiltradas` es un `computed()`, y un `computed()` solo se recalcula cuando cambia un signal que leyó adentro — con una propiedad plana (`filtros.tipo = x`), el cambio nunca invalidaba el caché y la tabla se quedaba mostrando el resultado viejo hasta que algún **otro** signal cambiara. Es un bug real que se evitó de raíz al construir el filtro de Tienda/Ecosistema, no una optimización preventiva.

Con tiendas o ecosistemas seleccionados en el filtro, las tarifas Genéricas se ocultan a propósito — no pertenecen a ninguna tienda puntual, así que filtrar por tienda y seguir viendo la Genérica sería confuso.

---

## Cobertura Genérica — un resumen honesto

Por cada tipo de liquidación (Entrega/Devolución), la pantalla muestra si existe una tarifa Genérica configurada. Si no la hay, cualquier tienda sin tarifa propia de ese tipo se queda **sin forma de cobrar ese flete** — es el tipo de vacío de configuración que conviene ver de un vistazo, sin tener que cruzar el catálogo de tiendas contra la tabla a mano.

También se muestra (colapsada por defecto) la lista de tiendas del catálogo que todavía no tienen ninguna tarifa propia para esta transportadora — dependen 100% de la Genérica.

---

## Formulario de tarifa: plata en formato colombiano

Los campos de dinero (flete, incrementos, mínimo de servicio) se muestran formateados (`$ 21.000`) y se editan como número plano al enfocar el campo — más fácil de editar que el texto formateado. El valor real que se guarda **nunca** lleva puntos ni símbolo de peso; `parseCOP` limpia cualquier carácter no numérico antes de guardar.

Un detalle de por qué la sincronización pasa en cada tecla (`(input)`) y no solo al perder el foco (`(blur)`): si el usuario tipeaba un flete y hacía click directo en "Guardar" sin que el navegador llegara a disparar `blur`, `guardarTarifa()` podía leer el número **viejo** y perder lo que se acababa de escribir. Ahora el valor real queda al día en cada tecla; `(blur)` solo reformatea lo que se ve en pantalla.

---

## Validaciones al guardar

- Si el alcance es `Tienda`, tiene que haber una tienda real seleccionada del catálogo.
- Tipo de liquidación obligatorio.
- Porcentajes en rango: recaudo y seguro entre 0 y 100; kilo adicional entre 0 y 300 (puede superar 100 en tarifas agresivas, pero un valor de miles sigue siendo un error de tipeo).
- Mínimo de servicio no negativo.
- **Sin duplicados**: no puede existir ya una tarifa con la misma combinación tienda(o Genérico)+tipo — si hubiera dos, el motor no sabría cuál usar.
- Ningún flete base ni incremento negativo.
- Al guardar, solo se conservan los trayectos con flete configurado (> 0) — una fila en cero significa "todavía sin definir", no "flete gratis".

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-09-16 | Iker Acevedo | Documentación inicial. |

---

## Observaciones

- El panel de detalle (`verDetalle`) muestra el total acumulado kilo a kilo — no el incremento suelto que se tipea en el formulario — para que quede claro cuánto vale el paquete en cualquier peso sin tener que sumar a mano.
- `editarDesdeDetalle()` corrige un bug real que existía en el HTML: cerrar el panel de detalle antes de abrir el de edición ponía `tarifaDetalle` en `null` antes de leerlo, así que el botón de "editar desde el detalle" no hacía nada visible. Ahora la referencia se guarda antes de cerrar.
