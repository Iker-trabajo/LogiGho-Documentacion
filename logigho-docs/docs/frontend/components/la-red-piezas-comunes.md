---

## Autor: Yadilton Torres

Fecha creacion: 2026-11-06
Estado: desarrollo
Tipo: componente

# Componente: piezas comunes de "La Red"

**Selector:** `app-logo-animado` (el resto son funciones y estilos)  
**Ubicación:** `src/app/views/pages/registro/la-red/form/avisos.ts`, `la-red/form/rutas.ts`, `src/app/components/logo-animado`, `views/pages/registro/styles/registro.scss`  
**Acceso:** Público

---

## ¿Qué hace?

Son las piezas que comparten el [registro](../views/pages/registro/registro.md), el [inicio de sesión](login.md) y recuperar contraseña: los avisos de lo que va pasando, el logo animado, los botones, el enlace a redes y las direcciones públicas.

---

## Avisos flotantes (`avisos.ts`)

```typescript
mostrarAviso(AVISOS.ingreso.verificando);          // en proceso: queda visible
mostrarAviso(AVISOS.ingreso.bienvenida, 'ok');     // éxito: se va a los 6 s
mostrarAviso(AVISOS.ingreso.rechazado, 'error');   // error: se va a los 4 s
ocultarAviso();
```

| Tipo | Icono | Duración |
|---|---|---|
| `proceso` | Anillo que gira | Hasta que otro aviso lo reemplace |
| `ok` | Check verde que cae, con barra de tiempo | 6 s |
| `error` | "!" rojo que se sacude, con barra de tiempo | 4 s |

- **Dónde sale:** siempre arriba a la derecha, bajo la barra del logo. En celular, arriba y a lo ancho, respetando la muesca.
- **No tapa nada:** mientras el aviso se ve, lo que flota debajo (la caja, "Tu ruta", la etiqueta y el sello de un país vecino, el camión) se desvanece con la clase `lared-tapado-por-aviso` y vuelve al irse. Se revisa cada 200 ms, porque la etiqueta de un vecino puede aparecer con el aviso ya visible.
- **Textos en un solo lugar:** todos están en `AVISOS` (registro, recuperación e ingreso). Regla: **ningún texto confirma si un correo tiene cuenta**.

---

## Logo animado (`app-logo-animado`)

- **Diseño:** un SVG hecho a partir de `Icons/Logoti.png`, totalmente blanco.
- **Entrada:** la flecha, el rombo y las letras entran en cascada, con un destello.
- **Clic:** las letras se recogen hacia el rombo, y otro clic las saca. Usa la API de animaciones para no reiniciar la entrada del CSS.
- **Dónde se usa:** en el login y el registro.

---

## Botones

Los estilos están al final de `registro.scss` y valen para login, registro y recuperar:

| Botón | Estilo |
|---|---|
| Principal (`.lared-btn--primary`) | Degradado naranja → rojo → frambuesa, relieve, resplandor que respira, destello al pasar el mouse y efecto de hundirse al presionar |
| Secundario (`.lared-btn--ghost`) | Vidrio con borde que se enciende en naranja |
| Barra de arriba (`.lared-topbar__boton`) | Borde con el degradado de la marca que se rellena al pasar el mouse |

En pantallas táctiles (`@media (hover: none)`) los efectos de pasar el mouse se cambian por un efecto al presionar, para que no queden pegados.

---

## "Síguenos en nuestras redes"

Va en el pie del login y del registro, en el lugar donde antes decía "© LogiGho":
- **Destino:** abre el Linktree `https://linktr.ee/logighosas` en otra pestaña.
- **Parpadeo:** de naranja a casi blanco con resplandor (`lared-redes-parpadeo`).
- **Con los países vecinos:** al pasar por uno, `MapaLaRed.brillarRedes()` lo hace parpadear al doble de velocidad.
- **En celular:** va como una píldora fácil de tocar.

---

## Direcciones públicas (`rutas.ts`)

```typescript
rutaPublica('login'); // '/login' en el servidor local · '/app/login' publicado
```

Las pantallas publicadas viven bajo `/app`, pero en el servidor local están en la raíz. Los enlaces escritos a mano (fuera del router) usan esta función.

---

## Pruebas unitarias

| Archivo | Qué cubre |
|---|---|
| `avisos.spec.ts` | Texto del aviso, tiempos de cada tipo, ocultar y devolver lo que tapa, y que ningún texto revele cuentas |
| `rutas.spec.ts` | Direcciones con y sin `/app` |

---

## Historial de cambios

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-11-06 | Yadilton Torres | Avisos flotantes, logo animado blanco, botones nuevos, "Síguenos en nuestras redes" y `rutaPublica`. |
