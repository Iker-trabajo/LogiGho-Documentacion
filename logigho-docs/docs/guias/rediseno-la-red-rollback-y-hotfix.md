# Rediseño «La Red» — plan de rollback y de hotfix

**Autor:** Iker Acevedo (decisión) · redactado con Claude · **Fecha de creación:** 2026-11-23 · **Estado:** plan escrito; **no ensayado**; el rediseño aún no está en producción

Qué hacer si algo falla después de liberar el [rediseño «La Red»](../frontend/tema-la-red.md): del ajuste en el navegador de una persona al rollback completo por git. Basado en `plan-rollback-y-hotfix.md` de la spec 021 (workspace), contrastado con el código el 2026-11-23.

!!! warning "Estado a 2026-11-23"
    - El rediseño está verificado en local (`45dbf949`) y **no está en PreProd ni en producción**. Los comandos de esta página son un plan: ninguno se ha ensayado.
    - Marcas: **observado** = lo leí en el repo o en un pipeline; **hipótesis** = no lo he comprobado.
    - Producción hoy: `origin/main` en `c32747b7` (**observado** el 2026-11-23; el día de la liberación puede ser otro: `git rev-parse origin/main`).

## 0. Por qué el rollback es seguro

- **Solo front** (**observado**): `git diff c32747b7..HEAD` solo tiene archivos de `SitioLogiGho/src` (638, todos `.scss`, `.html` o `.ts`). No toca Lambdas, base de datos, Terraform ni contratos de API.
- No hay migraciones ni datos nuevos: volver atrás no deja nada inconsistente. Lo único funcional que viaja en la rama es el filtro «Ecosistema» de Power BI (commit `0c88ca39`): es lógica del navegador y se va con el resto.
- Lo único que queda en el navegador de cada usuario son tres claves inofensivas: `lg-modo` y `lg-rendimiento` en `localStorage` y `lg-rendimiento-medido` en `sessionStorage`. La versión anterior las ignora.
- **Quitar `data-lg-tema` de `index.html` NO es una reversión válida** (`ENTREGA.md` §8): las vistas migradas consumen los tokens que ese atributo define y quedarían sin estilo. (`spec.md` §7 decía lo contrario; manda `ENTREGA.md`.)

## 1. Antes de fusionar a `main` (preparación, 5 minutos)

1. Anota el commit de producción vigente: `c32747b7` (o el de ese día).
2. En Vercel, anota el despliegue de producción vigente de **los dos proyectos** (**observado** en `azure-pipelines-cd-prod.yml`): uno para `logigho.com` y otro para `www.logigho.com`. Es a lo que se vuelve en el nivel 1.
3. Libera en un momento de poco uso y con alguien disponible la primera hora.

## 2. Qué hacer según lo que pase

| Síntoma | Acción | Tiempo |
|---|---|---|
| Una persona ve algo raro solo en su equipo (animaciones, modo claro) | Nivel 0: ajuste en su navegador | 1 min |
| Una pantalla o un botón concreto quedó mal o inalcanzable | Nivel 2: hotfix de esa vista | 30–60 min |
| Una función nueva molesta (precarga, transición, modo básico, interruptor, filtro de Power BI) | Nivel 3: apagar esa función con un hotfix de una línea | 30–45 min |
| La plataforma no carga, el menú no funciona o no se puede operar | Nivel 1: volver al despliegue anterior en Vercel, y después nivel 4 | 2–5 min |
| Hay varios fallos y no se ve el fondo | Nivel 1 y nivel 4 | 5 min + 45 min |

Los tiempos son estimaciones del plan (**hipótesis**: no se han cronometrado).

## Nivel 0 · Sin desplegar (soporte a una persona)

En la consola del navegador de la persona (más en [Modo básico y celular](../frontend/modo-basico-y-movil-la-red.md)):

| Qué | Comando |
|---|---|
| Quitar animaciones y efectos | `localStorage.setItem('lg-rendimiento','basico');location.reload()` |
| Volver a la versión azul | `localStorage.removeItem('lg-modo');location.reload()` |
| Volver a lo automático | `localStorage.removeItem('lg-rendimiento');sessionStorage.removeItem('lg-rendimiento-medido');location.reload()` |

## Nivel 1 · Volver al despliegue anterior en Vercel (lo más rápido)

**Hipótesis:** no se entró al panel de Vercel de LogiGho al redactar esto; se describe la función estándar de Vercel.

1. En cada uno de los dos proyectos de producción: *Deployments* → el despliegue anterior (el anotado en la sección 1) → *Promote to Production* (o *Instant Rollback*).
2. Comprueba `logigho.com` y `www.logigho.com` en una ventana privada.
3. **Importante:** esto **no cambia git**. `main` sigue teniendo el rediseño y el siguiente merge a `main` lo volvería a desplegar. Hay que hacer después el nivel 4 o un hotfix.

## Nivel 2 · Hotfix de una vista (lo más probable)

La lógica de las vistas no se tocó (salvo el filtro de Power BI), así que un fallo funcional casi seguro es de presentación: un control tapado, sin contraste o fuera de pantalla.

1. Localiza la diferencia comparando con producción, sin tocar servicios: `node tools/rediseno-la-red/explorar.js --base --ruta /app/<ruta>` frente a la misma orden sin `--base`. **Ojo:** ese script solo tiene la sintaxis comprobada; no se ha ejecutado con ventana (`ENTREGA.md` §6). Alternativa comprobada: las capturas con `capturar.js`.
2. Rama desde `main`: `hotfix/AAAA-MM-DD-descripcion`.
3. Arregla el SCSS o la clase de esa vista. Si hay prisa y la vista es crítica, se puede dejar esa vista como estaba en producción: `git checkout c32747b7 -- SitioLogiGho/src/app/views/<area>/<vista>/` (se verá con el aspecto anterior, en mesa clara, hasta rehacerla).
4. PR `hotfix/*` → `main` (**observado**: el gate del CI lo permite). Despliegue a producción con aprobación de Iker.
5. Sincroniza `main` → `preprod` con una rama `chore/*`.

## Nivel 3 · Apagar una función nueva con un hotfix de una línea

Cada función nueva se conecta en un solo punto (**observado** en el código el 2026-11-23). Mismo flujo de rama `hotfix/*` que el nivel 2.

| Función | Cómo se apaga | Archivo (bajo `SitioLogiGho/src/`) |
|---|---|---|
| Precarga de módulos | Quitar `withPreloading(PrecargaLaRed)` | `app/app.config.ts` |
| Indicador y transición entre módulos | Quitar el atributo `lgNavegacion` del `.wrapper` | `app/layout/default-layout/default-layout.component.html` |
| Detección automática del modo básico | Quitar `this.rendimiento.iniciar()` | `app/layout/default-layout/default-layout.component.ts` |
| Interruptor de modo claro | Quitar las 4 etiquetas `<app-lg-interruptor-tema>` (queda la versión azul; el modo claro aún podría activarse con `lg-modo` guardado: quitar también el script de `index.html` si hace falta) | `default-header.component.html`, `menu-movil.component.html`, `login.component.html`, `registro.component.html` |
| Todas las animaciones nuevas | Añadir al final de `scss/lared/_movimiento.scss`: `.wrapper *, .wrapper *::before, .wrapper *::after { animation-duration: 0s !important; transition-duration: 0s !important; }` | `scss/lared/_movimiento.scss` |
| Filtro «Ecosistema» de Power BI | Quitar la línea `...filtrosTablasTiendaPbi(tiendas)` del arreglo `filters` en las 5 vistas (tienda: dashboard, informe de guías, rendimiento; dropshipping: dashboard; logística: sin despacho) | Los `.ts` de esas 5 vistas |

La fila del filtro de Power BI **se añadió al redactar esta página**: el plan original (anterior al commit `0c88ca39`) no la incluía. La aclaración sobre el script de pre-pintado en la fila del interruptor también es de esta redacción (**hipótesis**: no se probó qué ocurre si solo se quitan las etiquetas).

Después de quitar una línea de TypeScript, corre los tests de esa carpeta: alguno comprueba que la pieza está conectada y habrá que ajustarlo en el mismo hotfix.

## Nivel 4 · Rollback completo por git

1. Rama `hotfix/AAAA-MM-DD-rollback-rediseno` desde `main`.
2. `git revert -m 1 <sha del merge de preprod en main>` (el merge es no fast-forward, por eso `-m 1`).
3. PR `hotfix/*` → `main`, despliegue con aprobación de Iker.
4. Sincroniza `main` → `preprod` con `chore/*`, o `preprod` volverá a traer el rediseño en la siguiente liberación.
5. Para volver a publicarlo cuando esté arreglado: revertir el revert (`git revert <sha del revert>`) en una rama `feature/*` y seguir el flujo normal. Un merge nuevo de la misma rama **no** trae de vuelta los cambios revertidos.

**Rollback a medias (quedarse con la versión azul aprobada):** revertir solo la fase 2 no es un comando único: son 18 commits sobre la etiqueta `rediseno-la-red-noche-v1` (`3a531b16`), más los posteriores. Lo práctico es una rama `hotfix/*` creada desde `main` en la que se deja el árbol igual que la etiqueta (`git checkout rediseno-la-red-noche-v1 -- SitioLogiGho/src`), con su build y sus tests. **Hipótesis:** no se ha ensayado; la etiqueta es **local** y habría que subirla antes. Esa versión no incluye el filtro de Power BI ni los detalles posteriores.

## Qué mirar las primeras 48 horas

1. iPhone con Safari: menú móvil, login, cambio de módulo, un modal.
2. Una acción que escribe por área: crear pedido, guardar perfil, aprobar o rechazar, una importación, un desembolso.
3. Menú por rol con un usuario de cada rol.
4. Tableros de Power BI y su filtro «Ecosistema».
5. Reportes de soporte con palabras como «no veo», «no me deja», «se quedó en blanco», «muy lento».

## Mejoras y cambios posteriores (cuando todo va bien)

Flujo normal: rama `feature/*` o `fix/*` desde `preprod`, PR a `preprod`, validación, liberación a `main` ([guía de validación y salida](rediseno-la-red-validacion-y-salida.md)). Reglas para tocar el tema sin romper nada: [Sistema visual](../frontend/sistema-visual-la-red.md#como-migrar-o-crear-una-vista) y [Modo claro](../frontend/modo-claro-la-red.md). Pendientes ya anotados: M-354 a M-387 en `backlog.md`.

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-11-23 | Iker Acevedo · redactado con Claude | Primera versión, a partir de `plan-rollback-y-hotfix.md`. Se añade el filtro de Power BI y se deja visible la discrepancia sobre `data-lg-tema`. |
