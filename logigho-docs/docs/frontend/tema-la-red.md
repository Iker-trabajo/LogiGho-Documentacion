---

## Autor: Iker Acevedo (decisión) · redactado con Claude

Fecha creacion: 2026-11-23
Estado: desarrollo (construido y verificado en local; pendiente de validar en PreProd)
Tipo: estándar de estilos

# Rediseño «La Red» del front: tema visual de toda la plataforma

El rediseño lleva la identidad del inicio de sesión y del registro «La Red» (fondo azul noche con retícula, panel oscuro con línea cálida, acción principal naranja-rojo) a **todo el front**, y añade modo claro, revisión para celular, animaciones y un modo básico para equipos limitados. **No cambia datos, permisos, cálculos ni peticiones** (con una salvedad declarada más abajo).

!!! warning "Estado a 2026-11-23: verificado en local, pendiente de PreProd"
    - Construido y verificado **en local** sobre el commit `45dbf949` de la rama `feature/rediseno-integral-la-red`.
    - **Aún no está en PreProd ni en producción.** Al redactar esta página la rama solo existía en el equipo de Iker.
    - Después de esa verificación se añadieron **dos commits** (ver [Qué cambió después de la verificación](#que-cambio-despues-de-la-verificacion)). El commit final puede cambiar todavía: otra sesión aplica detalles pedidos por Iker.
    - Cada página de esta sección dice qué está **verificado en local** y qué está **pendiente de validar en PreProd**.

!!! info "Cómo leer las marcas"
    **Observado** = comprobado en el archivo, el código o la herramienta citada. **Iker (fecha)** = decisión o explicación suya. **Hipótesis** = no comprobado. Las cifras vienen de `specs/021_rediseno_visual/ENTREGA.md` del workspace salvo que se cite otra fuente. Esa carpeta no es navegable desde este sitio: guarda el detalle fino (evidencias, capturas, decisiones D1–D60).

## Resumen para gerencia

| Pregunta | Respuesta |
|---|---|
| ¿Qué es? | Una capa visual nueva sobre las pantallas que ya existen: mismos datos, mismos permisos, mismas operaciones, otra apariencia. |
| ¿Qué verá la gente? | Una plataforma azul noche (por defecto) o clara a elección, con un interruptor junto a «Consultar guía». Pantallas ordenadas para celular y transiciones suaves al cambiar de módulo. |
| ¿Qué se probó? | Pruebas automáticas, comparación de peticiones contra producción en las 156 rutas, scroll en celular, contraste del modo claro y capturas en un motor tipo Safari. Todo en un equipo local, con datos sintéticos. |
| ¿Qué **no** se probó? | iPhone real y Safari real; las acciones que escriben (guardar, aprobar, desembolsar, importar) una por una; el menú con usuarios reales de cada rol; el interior de los tableros de Power BI; cuatro modales en modo claro. Se prueban en PreProd. |
| ¿Cuánto cuesta en rendimiento? | En escritorio, sin diferencia medible. En un teléfono lento (simulado), la carga inicial sube un 1 % y cambiar a un módulo ya visitado tarda entre 33 y 82 ms más, a cambio de que el menú responda al instante. |
| ¿Se puede volver atrás? | Sí, y rápido: es solo front, sin base de datos ni servicios. Volver al despliegue anterior en Vercel toma de 2 a 5 minutos. Ver [Rollback y hotfix](../guias/rediseno-la-red-rollback-y-hotfix.md). |
| ¿Qué falta decidir? | Ver [Decisiones abiertas](#decisiones-abiertas-de-iker). |

## Qué ve el usuario

| Antes (`main`) | Ahora (rediseño) |
|---|---|
| Interior con la paleta de CoreUI y fuentes del sistema | Interior azul noche con la identidad del login, tipografías Plus Jakarta Sans (títulos) e Inter (cuerpo y cifras) |
| Un solo aspecto | **Modo claro «Papel de ruta»** con interruptor en la cabecera, en el menú del celular, en el login y en el registro. Se recuerda entre visitas y no parpadea al recargar. Por defecto, noche. Ver [Modo claro](modo-claro-la-red.md) |
| 26 rutas que se salían de la pantalla en celular (a 390 px) | Esas 26 ya no se desbordan; tarjetas del muro, tabla compartida y vistas de todas las áreas corregidas. Ver [Modo básico y celular](modo-basico-y-movil-la-red.md) |
| Cambio de módulo sin señal visible hasta que carga | El enlace del menú se marca al instante, el contenido se atenúa, una línea cálida avanza bajo la cabecera y la vista nueva entra por partes. Ver [Movimiento y navegación](movimiento-y-navegacion-la-red.md) |
| El navegador descarga el código de un módulo al entrar | Además lo adelanta al pasar el puntero o tocar el menú, y en reposo solo para las áreas del menú del usuario |
| Mismo efecto en equipos limitados | **Modo básico** automático (o forzado) sin animaciones ni efectos caros. En teléfono y tableta no hay animaciones |

## Qué no cambia

- **Datos, permisos, rutas, guardas, formularios y cálculos.** Las plantillas existentes solo cambian atributos `class`, la inserción del interruptor y un atributo en el contenedor del layout (**observado**, `ENTREGA.md` §10).
- **Las peticiones al montar cada pantalla.** Comparadas contra `main` en las 156 rutas navegables, en noche y en claro: 0 diferencias en 1.144 peticiones (método, ruta, query, nombres de cabeceras, cuerpo y número de veces).
- **La escena 3D del login y el registro** (`views/pages/registro/la-red/**`): 0 líneas cambiadas (**observado**: `git diff` entre `c32747b7` y la cabeza actual de la rama sale vacío para esa carpeta).
- `package.json`, `package-lock.json`, `angular.json`, archivos de entorno y pipelines (`PR-preprod.md`).
- **No viene en esta rama:** el cobro de servicios (otra rama; al integrarla habrá que resolver conflictos en archivos que aquí cambian de estilo) y la landing pública.

!!! danger "Salvedad: un cambio funcional entró con el rediseño"
    El 2026-11-23, a pedido expreso de Iker, la rama incluye un cambio que **sí es lógica de vista**: en cinco vistas con tableros de Power BI (tienda: dashboard, informe de guías y rendimiento; dropshipping: dashboard; logística: sin despacho), el filtro «Ecosistema» solo lista los ecosistemas de las tiendas del usuario (commit `0c88ca39`, función `filtrosTablasTiendaPbi`). Con la opción «Todas» el reporte sigue sin filtros. No añade peticiones. **No está verificado contra el reporte real** (backlog M-387) y no estaba en el resumen del PR (`PR-preprod.md`, que cuenta 17 archivos TypeScript de producción; con este cambio son 23). Se valida en PreProd: ver la [guía de validación](../guias/rediseno-la-red-validacion-y-salida.md).

## Alcance

| Dato | Valor | Fuente |
|---|---|---|
| Superficies del inventario (componentes de la base `c32747b7`) | 332 | `ENTREGA.md` §2 |
| Migradas a noche y revisadas en captura | 291 | idem |
| Cimientos (shell, tabla, selectores, primitivas) | 22 | idem |
| Fuera del rediseño (login y registro de referencia, landing, 14 demos de CoreUI, una copia huérfana de wallet) | 19 | idem |
| Pendientes | 0 | idem |
| Rutas navegables revisadas con datos de prueba | 156, más login, recuperar y registro | idem |
| Archivos cambiados frente a `main` en `45dbf949` | 631 (343 SCSS, 256 plantillas, 32 TypeScript, 17 de producción) | `PR-preprod.md`; **observado** con `git diff --shortstat` |
| Archivos cambiados frente a `main` en la cabeza actual | 638 (343 SCSS, 256 plantillas, 39 TypeScript, 23 de producción) | **observado** (ver [Discrepancias](#discrepancias-entre-fuentes-y-codigo)) |

## Estado y verificación

Fechas: la versión noche (fase 1, commit `3a531b16`, etiqueta local `rediseno-la-red-noche-v1`) la probó y aprobó Iker el 2026-11-22. Ese mismo día pidió la fase 2 (modo claro, celular, animaciones, transición entre módulos) y autorizó construirla en local sin aprobaciones intermedias. La verificación final es del 2026-11-23, sobre `45dbf949`.

### Verificado en local (commit `45dbf949`)

| Prueba | Resultado | Fuente |
|---|---|---|
| Tests unitarios | 2.246 de 2.259 pasan. Los 13 que fallan son los mismos de `main` (11 nombres distintos) | `ENTREGA.md` §4 |
| Builds `preprod` y `production` | Correctos. Inicial 3,81 MB / 953 kB transferidos (`main`: 3,83 MB / 960 kB) | `ENTREGA.md` §4; `PR-preprod.md` |
| Peticiones al montar las 156 rutas, noche y claro | 0 diferencias frente a `main` | `ENTREGA.md` §4 |
| Scroll horizontal a 390 px | 130 rutas sin desborde, 26 que `main` desbordaba y ya no, 0 nuevas | `ENTREGA.md` §4 |
| Scroll horizontal a 768 px | 155 sin desborde, 1 corregida, 0 nuevas | `ENTREGA.md` §4 |
| Contraste del modo claro | 60 de 60 pares de tokens y 27 de 27 del login en su umbral | `ENTREGA.md` §4 |
| Pre-pintado del modo claro | Con el modo guardado, `<html>` ya trae `data-lg-modo="dia"` antes de arrancar Angular | `ENTREGA.md` §4 |
| Capturas del build final | 159 pantallas × 3 anchos × 2 modos, ninguna fallida | `ENTREGA.md` §4 |
| Motor WebKit (el de Safari), humo | 34 capturas de pantallas clave, sin errores de JavaScript ni pantallas rotas | `PR-preprod.md`; `orquestacion.md` |
| Versión noche frente a la aprobada, 1440 px | 105 de 156 rutas idénticas píxel a píxel; el resto, el cambio pedido en el muro y suavizado de texto | `ENTREGA.md` §4 |
| Revisiones independientes | 4 (R1 a R4); la última, 0 bloqueantes y 0 altos tras las correcciones | `ENTREGA.md` §4 |
| Fusión | La rama contiene todo `preprod` y todo `main`: entra sin conflictos | `PR-preprod.md` |

!!! note "Quién hizo las revisiones"
    Las revisiones R1 a R4 las hizo **un modelo de IA distinto del que escribió el código**, no personas (`orquestacion.md`). Son una barrera útil, no sustituyen la prueba humana en PreProd. La spec menciona además una auditoría externa con otro modelo (Codex) como pendiente; no consta su resultado.

### Pendiente de validar en PreProd (no verificado)

| Qué | Por qué falta |
|---|---|
| **iPhone real y Safari real** | Todo se probó en Chromium con emulación y un humo en WebKit de escritorio. El movimiento usa `translate` y `scale` como propiedades sueltas y `:has()` |
| **Acciones que escriben** (guardar, aprobar, desembolsar, importar) una por una | La paridad automática cubre las peticiones al cargar cada pantalla. Aquí la garantía es indirecta: ninguna lógica de vista cambió (salvo el filtro de Power BI), plantillas con solo `class` y tests en verde |
| **Menú por rol con usuarios reales** | Se probó con un catálogo sintético derivado del router; el real vive en la colección `module` de la base de datos |
| **Interior de los marcos de Power BI** y el filtro «Ecosistema» | En las pruebas automáticas los marcos salen vacíos |
| **Cuatro modales en modo claro** | No abrieron con datos de prueba: estados de usuario, novedades de wallet y de inversionista, actualización de productos. También los pasos 2 a 4 de «Accesos» y un aviso real del registro |
| Modo claro del login con **GPU real** | Solo se midió con WebGL por software (M-381) |
| Desplegables de CoreUI sobre Popper, `:active` de botones y el barrido de la línea en SweetAlert | Movimiento sin probar en navegador |

### Costes medidos en un teléfono lento (simulado)

Chromium sin interfaz con CPU ×4 y red de 1,6 Mb/s, 5 repeticiones intercaladas. Es un equipo emulado, no un teléfono real ni Safari (`ENTREGA.md` §5).

| Qué | Noche aprobada | Final | Diferencia |
|---|---|---|---|
| Carga inicial hasta utilizable (login · wallet · tabla) | 20,30 · 29,30 · 26,97 s | 20,63 · 29,59 · 27,32 s | +0,30 a +0,36 s (**+1 %**) |
| Cambio a un módulo ya visitado | 95–277 ms | 128–359 ms | **+33 a +82 ms** |
| Toque en el menú hasta la primera respuesta visible (primera visita) | 0,38–8,5 s | 0,04–0,28 s | Mejora: responde antes |

En escritorio (carga, saltos de contenido, fluidez al desplazar y memoria) no hay diferencia fuera del ruido; la respuesta al clic en el menú baja de 48–147 ms a 19–31 ms.

## Qué cambió después de la verificación

La cabeza de la rama (`0c88ca39`, 2026-11-23) tiene dos commits sobre el verificado. **No consta que la verificación completa se haya repetido sobre ellos**; la regla del proyecto (D58) exige repetirla si cambia el código.

| Commit | Qué cambia |
|---|---|
| `9b7c5adb` | Solo SCSS de «Pedidos»: los botones «Acciones» y «Crear Pedidos» se expanden en 380 ms (antes 120 ms) y el logo se ve en modo claro. Noche intacta |
| `0c88ca39` | El filtro de Power BI descrito en la salvedad de arriba (lógica de vista, con su spec de 3 casos) |

## Decisiones abiertas de Iker

Tomadas por quien construyó el rediseño y a la espera de su decisión (`ENTREGA.md` §7). Detalle de las que tienen ADR en [Decisiones del rediseño](../arquitectura/adr/rediseno-la-red/index.md).

| Tema | Lo que se hizo | Alternativa |
|---|---|---|
| Modo por defecto | Noche; no se sigue la preferencia del sistema operativo | Seguir `prefers-color-scheme` |
| Navegador a 30 cuadros por segundo (ahorro de energía, escritorio remoto) | Entra en modo básico | Subir el umbral para que conserve animaciones |
| Coste en teléfono lento | +1 % de carga y +33 a +82 ms al volver a un módulo, a cambio de respuesta inmediata | Quitar la precarga o el indicador en teléfono |
| Muro, columna lateral en escritorio | Ya no acompaña al scroll (era la causa del corte y del doble scroll de «Comunidades» y «Miembros») | Volver a fijarla con un solo scroll de columna |
| Muro en teléfono | Las tarjetas laterales siguen ocultas, como en `main` | Mostrarlas (M-376) |
| Línea de progreso al cambiar de módulo | Con cargas de 120–300 ms se vacía y se rellena muy rápido | Retrasar su aparición o quitarla en cargas cortas |
| Login claro | Mapa con mezcla y filtros CSS sobre el lienzo, sin tocar la escena | Paleta clara nativa de la escena (TypeScript, M-381) |
| `/app/registercontact` en claro | Su tarjeta pasa a blanca | Fijarla en azul marino (M-379) |
| SweetAlert2 | Claro en los dos modos | Versión noche (79 llamadas llevan colores fijos) |
| Landing pública | Sin tocar | Rediseñarla en otra historia (M-371) |

## Discrepancias entre fuentes y código

Se dejan visibles como pide `AGENTS.md`:

1. **Archivos cambiados.** `PR-preprod.md` dice 631 archivos y 17 de TypeScript de producción (cierto en `45dbf949`). En la cabeza actual son 638 y 23. El texto del PR sigue sin la salvedad de Power BI.
2. **Cómo revertir.** `spec.md` §7 dice que quitar `data-lg-tema` de `index.html` revierte el tema; `ENTREGA.md` §8 dice que **no** es una reversión válida (las vistas migradas consumen tokens que ese atributo define). Manda `ENTREGA.md`: ver [Rollback](../guias/rediseno-la-red-rollback-y-hotfix.md).
3. **Núcleos y memoria en el modo básico.** Las fuentes dicen «≤ 2 GB o ≤ 2 núcleos». En el código los núcleos solo cuentan si el navegador también expone la memoria (Chromium). Ver [Modo básico](modo-basico-y-movil-la-red.md).
4. **Interruptor bajo 375 px.** El diseño preveía ocultarlo en la cabecera por debajo de 375 px; no encontré esa regla en el SCSS de la cabecera. Qué pasa a esos anchos no está verificado.
5. **Número de la historia.** En `backlog.md` el ID 021 es la historia de autorización de rutas; la carpeta `specs/021_rediseno_visual` conserva el nombre que le dio Iker y su historia es la 035.
6. **Node.** El proyecto pide 22.x y la máquina de pruebas tiene 24.19.0 (misma versión para todas las corridas; el CI usa la 22).

## Mapa de esta sección

| Página | Qué responde |
|---|---|
| **Esta página** | Qué es, qué cambia y qué no, estado, cifras |
| [Sistema visual y tokens](sistema-visual-la-red.md) | Cómo está hecho; cómo migrar o crear una vista |
| [Modo claro e interruptor](modo-claro-la-red.md) | Paleta «Papel de ruta», activación, login en claro |
| [Movimiento y navegación entre módulos](movimiento-y-navegacion-la-red.md) | Catálogo de animaciones, directiva `lgNavegacion`, precarga |
| [Modo básico y celular](modo-basico-y-movil-la-red.md) | Equipos limitados, comportamiento en teléfono, soporte |
| [Validación en PreProd y salida a producción](../guias/rediseno-la-red-validacion-y-salida.md) | Lista de comprobación y criterio |
| [Rollback y hotfix](../guias/rediseno-la-red-rollback-y-hotfix.md) | Cómo volver atrás y qué vigilar |
| [Decisiones (ADR)](../arquitectura/adr/rediseno-la-red/index.md) | Por qué se eligió cada diseño |
| [Login, recuperación y cambio de contraseña](components/login.md) · [Registro «La Red»](views/pages/registro/registro.md) · [Piezas comunes](components/la-red-piezas-comunes.md) | Las pantallas de origen de la identidad |

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-11-23 | Iker Acevedo · redactado con Claude | Primera versión: estado, alcance, verificación y decisiones abiertas del rediseño, en local y pendiente de PreProd. |
