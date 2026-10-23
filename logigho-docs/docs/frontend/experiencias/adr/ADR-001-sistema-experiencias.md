---

## Autor: Iker Acevedo
Fecha creacion: 2026-10-17
Estado: propuesta

# ADR-001: Sistema de Experiencias (varios frontends visuales sobre una sola lógica)

**Autor:** Iker Acevedo
**Fecha:** 2026-10-17
**Estado:** Propuesta

---

## Contexto

Negocio quiere que LogiGho tenga **varias experiencias de usuario completas** (al menos tres, probablemente más).
Cada una cambia la organización de la navegación, la distribución de las vistas, los componentes visuales y los
colores. Como si fuera otro frontend, pero con la misma lógica, las mismas rutas y el mismo backend.

Estado actual observado en el código (2026-10-17):

| Hecho | Evidencia |
|---|---|
| Vistas monolíticas: HTTP, reglas, permisos y HTML en un solo componente | `tienda/pedidos/pedidos.component.ts`: 1.555 líneas; HTML 23 KB; SCSS 22 KB con 82 colores fijos |
| El layout mezcla lógica y presentación | `default-layout.component.ts`: carga usuario, módulos, valida acceso a rutas, alerta tiendas inactivas y arma el menú |
| Colores escritos a mano | 6.020 hexadecimales en 308 archivos SCSS; 3.450 usos de `var(--logigho-*)` |
| Ya existe el patrón Repository + Store | `tienda/gestion-cuentas`, `logistica/ingreso-devoluciones` |
| No hay herramientas que hagan cumplir reglas | Sin ESLint ni Stylelint; pruebas con Karma + Jasmine |

---

## Opciones consideradas

### Opción A: Un proyecto Angular por experiencia

**Pros:** independencia total entre equipos.
**Contras:** la lógica se duplica N veces; N despliegues, N correcciones por cada bug; las experiencias divergen.

### Opción B: Un solo componente por vista con `@if (experiencia === ...)`

**Pros:** no duplica lógica.
**Contras:** archivos que crecen sin límite; todas las experiencias se descargan siempre; conflictos constantes
entre desarrolladores; el SCSS de una experiencia afecta a otra.

### Opción C: Variantes elegidas vista por vista

Cada vista ofrece versiones y el usuario elige una por vista.

**Pros:** flexible.
**Contras:** no produce una experiencia coherente; las combinaciones crecen sin control (vistas × versiones); no
cubre la navegación ni el shell.

### Opción D: Micro-frontends (Module Federation)

**Pros:** despliegue independiente por experiencia.
**Contras:** complejidad de infraestructura y versionado desproporcionada para un equipo y un producto; la lógica
compartida igual hay que resolverla.

### Opción E: Experiencias globales sobre un cerebro compartido, con respaldo

La lógica vive una sola vez (cerebro de la plataforma y de cada vista). Cada experiencia aporta shell, organizador de
menú, tokens, UI kit y versiones de vistas, todo cargado de forma diferida. Si una vista no tiene versión para la
experiencia activa, usa la de `clasica`.

**Pros:** DRY real; una experiencia nueva no toca la lógica; se migra vista por vista sin bloquear a nadie;
solo se descarga la experiencia en uso; las URLs, permisos y datos son idénticos en todas.
**Contras:** requiere extraer la lógica del layout y de cada vista antes de su primera versión nueva; más
carpetas por vista; el equipo debe aprender las reglas de dependencias.

---

## Decisión

**Se elige:** Opción E, con el nombre **Sistema de Experiencias**.

**Razón:** es la única opción que escala a N experiencias sin multiplicar la lógica, permite migrar
progresivamente gracias al respaldo y mantiene un solo producto para desplegar, probar y asegurar.

---

## Consecuencias

**Positivas:**

- Crear una experiencia nueva es trabajo de diseño y presentación, no de lógica.
- La lógica se prueba con pruebas unitarias sin depender de ninguna experiencia.
- Los cambios de seguridad en el backend solo tocan los `repository`, nunca las experiencias.
- El respaldo permite publicar una experiencia nueva antes de que todas las vistas la tengan.

**Negativas:**

- Refactor inicial del layout y de cada vista que se migre.
- Se agregan ESLint y Stylelint para hacer cumplir las reglas; el equipo debe adoptarlos.
- Durante la migración, una experiencia nueva mostrará vistas en respaldo.

---

## Impacto en el código

| Módulo / Repo | Cambio |
|---|---|
| `SitioLogiGho` | `core/experiencias/` (motor), `core/plataforma/` (cerebro del shell), `experiencias/<id>/` (shell, navegación, estilos, UI kit) y `views/**/experiencias/<id>/` |
| `SitioLogiGho` | ESLint con reglas de dependencias; Stylelint sin colores fijos en código nuevo |
| `LambdasLogiGho` | Lambda de preferencias del usuario (experiencia elegida), con las reglas de [seguridad por diseño](../seguridad-por-diseno.md) |
| `Repo-LogiGho-Documentacion` | Esta sección y una ficha por vista con la [plantilla](../../../plantillas/PLANTILLA-VISTA-EXPERIENCIAS.md) |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-10-17 | Iker Acevedo | Creación de la propuesta |

---

## Referencias

- [Sistema de Experiencias](../index.md)
- Precedente del patrón Strategy en backend: `ADR-001-strategy-cotizadores.md`
