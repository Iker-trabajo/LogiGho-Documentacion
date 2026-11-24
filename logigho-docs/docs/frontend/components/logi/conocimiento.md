---
autor: Yadilton Torres
fecha_creacion: 2026-11-24
ultima_actualizacion: 2026-11-24
estado: desarrollo
nivel: 2
---

# Logi: conocimiento

**Autor:** Yadilton Torres

**Ubicación:** `SitioLogiGho/src/app/components/logi-pet/conocimiento/`

Solo textos y datos: lo que Logi sabe de LogiGho. Todo es local, sin IA ni servidor.

---

## Archivos

| Archivo | Qué tiene | Funciones |
|---|---|---|
| `logi-conocimiento.ts` | ~140 pantallas (`CONOCIMIENTO`), con para qué sirven (`que`), consejos y errores típicos de esa pantalla | `pantallaDe(ruta)`: busca la pantalla; las rutas con id usan su parte fija. `consejoDe(ruta)`: un consejo distinto cada vez |
| `logi-guias.ts` | 39 guías de «Pregúntale a Logi» (`GUIAS`): pregunta, palabras con que la gente lo pregunta, resumen, pasos, `ruta` y tema | `responder(texto, maximo)`: las guías que mejor responden. `guiasDePantalla(ruta)`. `guiaPorId(id)` |
| `logi-tips.ts` | Consejo al entrar a una pantalla | `tipPantalla(url, nombre)`: consejo de la pantalla, tip por ruta o frase del menú |
| `logi-ayuda-menu.ts` | Frase de cada opción del menú, por ruta y por nombre | `ayudaMenu()`, `ayudaEnlaceMenu()`, `normalizarRuta()`, `normalizarNombre()` |
| `logi-botones.ts` | Frase de los botones importantes (campana, día/noche, Exportar, Desembolso…) | `ayudaBoton(boton)` |

---

## Cómo agregar

- **Una guía:** sumar un elemento a `GUIAS`. `palabras` van sin tildes y en minúsculas; entre más sinónimos, mejor la encuentra. Si la guía es de una pantalla, su `ruta` va **sin `/app`** y debe ser la misma del módulo en la colección `module` (si no, no le sale a nadie; ver [Seguridad por rol](seguridad-por-rol.md)).
- **Una pantalla:** sumar la ruta (sin `/app`, en minúsculas) a `CONOCIMIENTO` con `que`, `consejos` y, si hace falta, `errores` propios.
- **Un botón:** sumar una regla a `logi-botones.ts` con la expresión que reconoce su texto o `aria-label`.

---

## Búsqueda

`responder()` suma puntos por frases completas (pesan más) y por palabras sueltas, con tolerancia a plurales y palabras a medio escribir. Solo devuelve guías con suficiente puntaje. El componente filtra después las que el rol no puede ver.

---

## Pruebas

- `logi-guias.spec.ts`: ids únicos; rutas sin `/app`, en minúsculas y sin `/` final; cada guía con pregunta, resumen y pasos; Ingreso Devoluciones en `/logistica/devolucion`; búsqueda sin tildes; máximo de respuestas.
- `logi-conocimiento.spec.ts`: claves sin `/app` y en minúsculas; cada pantalla dice para qué sirve; rutas con id; rotación de consejos.
- `logi-ayuda-menu.spec.ts`: normalización de rutas y nombres; frase por ruta y por grupo; `null` si no la conoce.
