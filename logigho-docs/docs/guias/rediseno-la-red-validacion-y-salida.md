# Rediseño «La Red» — validación en PreProd y salida a producción

**Autor:** Iker Acevedo (decisión) · redactado con Claude · **Fecha de creación:** 2026-11-23 · **Estado:** pendiente: el rediseño aún no está en PreProd

Guía para validar el [rediseño «La Red»](../frontend/tema-la-red.md) en PreProd y decidir si pasa a producción. Pensada para que gerencia sepa qué se está aprobando y para que quien valide tenga una lista concreta.

!!! warning "Estado a 2026-11-23: verificado en local, pendiente de PreProd"
    - Verificado en local sobre el commit `45dbf949` de `feature/rediseno-integral-la-red`. La cabeza de la rama ya tiene dos commits más (`9b7c5adb` y `0c88ca39`) y puede cambiar: la regla del proyecto es repetir la verificación completa si el código cambia.
    - El veredicto vigente es «**listo para PreProd**», no «listo para producción» (decisión D59): a producción se llega desde `preprod`, y lo que falta solo se puede probar desplegado.
    - Cifras y pendientes: [Rediseño «La Red»](../frontend/tema-la-red.md). Marcas: **observado**, **Iker (fecha)**, **hipótesis**.

!!! danger "Reglas de oro"
    1. **Nada va a producción sin instrucción explícita de Iker.** PR, merge y despliegues los hace Iker (Iker, 2026-11-21); la IA solo empuja ramas `feature/*`.
    2. Las pruebas en PreProd se hacen con **datos de prueba**: algunos destinos de terceros son los mismos que en producción (`ENTREGA.md` §1). Una acción de desembolso, importación o cobro con datos reales es real.
    3. Si un criterio de la lista no se cumple, **no se avanza**: se corrige o se decide qué se acepta.

## Resumen para decidir

| Se aprueba | Se sabe | No se sabe todavía |
|---|---|---|
| Un cambio solo de front (631 → 638 archivos, sin base de datos, servicios ni infraestructura) con apariencia nueva, modo claro, celular corregido, animaciones y modo básico | Mismas peticiones al cargar las 156 pantallas, 2.246 de 2.259 pruebas (las 13 que fallan fallan igual en `main`), builds correctos, sin desborde nuevo en celular, contraste del modo claro, humo en motor WebKit, volver atrás en 2 a 5 minutos | Cómo se ve y se siente en un iPhone real; si cada acción que escribe funciona (guardar, aprobar, desembolsar, importar); el menú con usuarios reales de cada rol; los tableros de Power BI y su filtro nuevo; cuatro modales en claro |

Costes: +1 % de carga inicial y +33 a +82 ms al volver a un módulo, solo en un teléfono lento simulado; sin diferencia en escritorio.

## Flujo de ramas

```
feature/rediseno-integral-la-red  →  PR a preprod  →  (validación en PreProd)  →  preprod  →  main  →  producción
```

| Paso | Quién | Detalle |
|---|---|---|
| 1. Publicar la rama | IA (solo ramas `feature/*`) | Con instrucción expresa de Iker (D60): hasta ahora «todo queda en local». Se entrega el enlace del PR |
| 2. PR `feature/*` → `preprod` | Iker | El CI de PR (`azure-pipelines-ci.yml`) exige la regla de ramas (`feature/*` → `preprod` está permitido), `npm ci` y el build de producción; **los tests son informativos** (**observado**). Verificado: la rama contiene todo `origin/preprod` y todo `origin/main`, entra sin conflictos |
| 3. Validación en PreProd | Iker y quien él designe | La lista de abajo |
| 4. Liberación `preprod` → `main` | Iker | Merge no fast-forward. A `main` solo se llega desde `preprod` o desde una rama `hotfix/*` |
| 5. Despliegue a producción | Iker | Con su aprobación explícita y, idealmente, en un momento de poco uso y con alguien disponible la primera hora |
| 6. Sincronizar | Iker | Si hubo hotfix, `main` → `preprod` con una rama `chore/*` |

**Antes del PR:** si el commit final cambia, repetir la verificación (en el workspace, `herramientas/verificacion-final.sh verificar`: tests, builds, capturas, paridad y anchos con la red bloqueada) y actualizar las cifras de [Rediseño «La Red»](../frontend/tema-la-red.md). Node: el CI usa la 22 y las pruebas locales corrieron con la 24.19.0; no se replicó el Node 22.

**Aviso de integración:** la rama de cobro de servicios tocará archivos que aquí cambian de estilo; al integrarla habrá que resolver esos conflictos. El build emite 581 avisos de Sass por declaraciones tras reglas anidadas (`main`: 18); no bloquean, pero hay que ordenarlos antes de subir de versión de Sass (M-385).

## Lista de comprobación en PreProd

Marca cada fila con quién la probó, en qué equipo y qué vio. El detalle fino de los contratos de las acciones críticas está en `contratos/contratos-criticos.md` de la spec (workspace).

| # | Qué comprobar | Cómo | Resultado esperado |
|---|---|---|---|
| 1 | **iPhone real con Safari:** menú móvil, interruptor de tema, login claro, cambio de módulo y un modal | En un iPhone, con el sitio de PreProd | El menú abre y navega; el interruptor cambia el modo y se recuerda; el login claro se ve y no se cuelga; el cambio de módulo responde; el modal abre y cierra. Sin pantalla en blanco ni toques bloqueados. Es la prueba más importante: el último hotfix de producción fue del menú en iPhone y aquí solo hubo motores de escritorio |
| 2 | **Acciones que escriben, una por área:** crear un pedido, guardar un perfil, aprobar o rechazar, una importación, un desembolso de prueba | Con datos de prueba, comparando con el comportamiento actual | Mismas confirmaciones, mismos mensajes y mismo resultado que en producción hoy. No hay un solo botón que ya no responda |
| 3 | **Menú por rol** | Con un usuario real de cada rol (el catálogo real vive en la colección `module` de la base; se probó solo con uno sintético) | Cada rol ve su menú completo y navega a cada entrada (M-361) |
| 4 | **Marcos de Power BI**, noche y claro | Abrir cada tablero | Cargan y se ven bien en los dos modos (en las pruebas automáticas salían vacíos) |
| 5 | **Filtro «Ecosistema» de Power BI** (cambio funcional, commit `0c88ca39`) | Con un usuario con tiendas asignadas y con uno con «Todas», en tienda (dashboard, informe de guías, rendimiento), dropshipping (dashboard) y logística (sin despacho) | Con tiendas asignadas, el filtro «Ecosistema» solo lista los ecosistemas de sus tiendas; con «Todas», el reporte sigue sin filtros (M-387; sin verificar contra el reporte real) |
| 6 | **Modo claro en los modales que no se vieron** | Abrir y revisar: estados de usuario, novedades de wallet y de inversionista, actualización de productos, pasos 2 a 4 de «Accesos» | Texto legible y controles con borde visible (M-380) |
| 7 | **Modo claro en login, recuperar y registro** con GPU real | En un equipo con GPU y en un celular | Se ve y se mueve con fluidez (M-381) |
| 8 | **Modo básico** | Forzarlo con la consola ([comandos](../frontend/modo-basico-y-movil-la-red.md)) y recorrer tres módulos | Sin animaciones, nada invisible ni congelado, los indicadores de carga siguen girando |
| 9 | **Rendimiento en teléfono real** | Carga inicial y volver a un módulo ya visitado | Sin sensación de lentitud frente a producción. Las cifras de referencia son simuladas |
| 10 | **Cambio de módulo** en escritorio y celular | Navegar entre cuatro módulos varias veces | El enlace se marca al instante; la línea cálida avanza; nada queda atenuado ni con la línea a medias |
| 11 | **Impresión y exportaciones** de una vista que imprime desde el DOM | Imprimir | Texto oscuro sobre blanco en noche y en claro |
| 12 | **Desplegables, `:active` de botones y SweetAlert** | Usarlos | Se ven bien (movimiento no probado en navegador) |
| 13 | **Un navegador a 30 fps o de 2 núcleos** | Si hay uno a mano | Si entra en modo básico, es el comportamiento declarado; confirmar con Iker si molesta |

## Criterio para pasar a producción

Se pasa a producción solo si **todo esto es cierto** y Iker lo avala:

1. Las filas 1 a 6 de la lista están probadas en PreProd y sin defectos que impidan operar. Las filas 7 a 13 se probaron o Iker aceptó explícitamente el riesgo de no probarlas.
2. La verificación completa se repitió sobre el commit que se libera y las cifras publicadas son de ese commit.
3. Iker decidió (o aceptó los valores por defecto de) las [decisiones abiertas](../frontend/tema-la-red.md#decisiones-abiertas-de-iker): modo por defecto, 30 fps, coste en teléfono, muro, línea de progreso, login claro, `registercontact`, SweetAlert y landing.
4. Está anotado el despliegue vigente de producción de **los dos proyectos de Vercel** (uno para `logigho.com` y otro para `www.logigho.com`) y el commit de `main` de ese día, para poder volver atrás ([Rollback](rediseno-la-red-rollback-y-hotfix.md)).
5. Hay alguien disponible la primera hora y se sabe a quién se avisa si soporte reporta «no veo», «no me deja», «se quedó en blanco» o «muy lento».

Si falla un criterio: se corrige en una rama `fix/*` desde `preprod` (nunca directo a `main`), se repite la verificación y se vuelve a validar.

## Después de la salida

Vigilar las primeras 48 horas según [Rollback y hotfix](rediseno-la-red-rollback-y-hotfix.md#que-mirar-las-primeras-48-horas). Después, el flujo normal: rama `feature/*` o `fix/*` desde `preprod`, PR a `preprod`, validación, liberación a `main`. Las reglas para tocar el tema sin romper nada están en [Sistema visual](../frontend/sistema-visual-la-red.md#como-migrar-o-crear-una-vista).

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-11-23 | Iker Acevedo · redactado con Claude | Primera versión. El rediseño sigue en local, pendiente de PreProd. |
