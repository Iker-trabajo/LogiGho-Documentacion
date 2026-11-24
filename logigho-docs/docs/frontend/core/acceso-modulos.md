---
autor: Yadilton Torres
fecha_creacion: 2026-11-24
ultima_actualizacion: 2026-11-24
estado: desarrollo
nivel: 2
---

# Acceso a módulos (seguridad de rutas)

**Autor:** Yadilton Torres

**Ubicación:** `SitioLogiGho/src/app/core/application/services/guards/acceso-modulos.ts`

**Rama:** `feature/logo-animado` (commit `553a7b71`).

---

## ¿Qué hace?

Una sola regla para saber si el usuario puede abrir una ruta según la colección `module`. La usan:

- El [layout](../layout/default-layout.md) en `validateModuleAccess()`: si no puede, lo manda a login.
- [Logi](../components/logi/seguridad-por-rol.md): no muestra guías, preguntas ni consejos de lo que no puede usar.

---

## El hueco que cerró

Antes, el layout daba acceso si la ruta **empezaba** por la ruta de cualquier módulo del usuario. Un módulo padre abría todas sus hijas aunque tuvieran sus propios roles:

| Rol | Pantalla | Antes | Ahora |
|---|---|---|---|
| Tienda | `/app/tienda/rechazados`, `/app/tienda/flujo-masivo-guias` | Entraba (por Mi Tienda `/app/tienda`) | Bloqueado |
| Jefe Logística | `/app/administracion/permisos` | Entraba (por Administración) | Bloqueado |
| Jefe Logística | `/app/administracion/tiendas` (suya) | Entra | Entra |
| Todos | Inicio y Muro | Entra | Entra |

---

## Regla

1. La ruta se normaliza: sin `/app`, sin `?query` ni `#hash`, sin `/` final y en minúsculas.
2. **Rutas libres** (`RUTAS_LIBRES`): inicio, `/muro-compartido` y `/maps` abren para cualquiera con sesión.
3. Se busca el módulo **ACTIVO más específico** cuya `ruta` contiene a la pedida (prefijo más largo, cortando en `/`).
4. Se permite solo si el usuario tiene un módulo con esa misma ruta. Varios módulos pueden compartirla (ej. Mi Tienda y Pedidos Masivos): basta con uno.
5. Si **ningún** módulo la cubre, se usa la regla anterior (empieza por la ruta de un módulo del usuario). La necesitan las pantallas `/controller/*` sin módulo propio, que entran por «Controller» (ruta `/app/controll` en la colección).

```ts
rutaPermitida('/app/tienda/rechazados', todos, delUsuario) // false para Tienda
rutaPermitida('/app/tienda/detalle/5', todos, delUsuario)  // true: cuelga de Mi Tienda
```

---

## API

| Elemento | Qué hace |
|---|---|
| `normalizarRutaModulo(ruta)` | Normaliza como en el paso 1 |
| `esRutaLibre(ruta)` | `true` para inicio, Muro y Maps |
| `rutaPermitida(ruta, todos, delUsuario)` | La regla. `todos` = módulos ACTIVO sin filtrar; `delUsuario` = los de sus roles (o con `Todos`) |
| `AccesoModulosService.registrar(todos, delUsuario)` | Lo llama el layout al cargar los módulos |
| `AccesoModulosService.puede(ruta)` | `false` mientras no carguen los módulos |

---

## Cambios en el layout

- `fetchModules()` guarda `todosLosModulos` (antes de filtrar por rol) y llama a `accesoModulos.registrar(...)`.
- `validateModuleAccess()` usa `esRutaLibre()` y `rutaPermitida()` en lugar de comparar con `startsWith`.

---

## Pruebas

`acceso-modulos.spec.ts`, con módulos reales de la colección:

- Tienda entra a sus pantallas y a los detalles de Mi Tienda, pero no a Rechazados ni a Flujo Guías.
- Jefe Logística entra a Tiendas pero no a Permisos.
- Respaldo de `/controller/*`, ruta sin módulo, Muro sin módulos y `/tienda` frente a `/tiendas`.
- El servicio responde `false` hasta que carguen los módulos.

`default-layout.component.spec.ts` agrega: deja pasar a una pantalla propia, saca de una hija con roles propios (va a `/login` y marca `accesoDenegado`), Muro abierto y módulos entregados a Logi.

---

## Observaciones

- Las pantallas de módulos **INACTIVO** (ej. Cotizador, Mi Score) siguen abriendo a mano para quien tenga el módulo padre, porque el layout solo consulta `module?estado=ACTIVO`.
- Lo correcto es arreglar la ruta del módulo «Controller» (`/app/controll` → `/app/controller`) en la colección; entonces el respaldo del paso 5 deja de hacer falta para esas pantallas.
- La validación corre después de navegar (`NavigationEnd`): la pantalla alcanza a cargar antes de sacar al usuario. El backend debe seguir validando el rol en cada servicio.
