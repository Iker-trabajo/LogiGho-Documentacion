# Módulo: Administrar Comunidades

---

## Autor: Adalberto González
Fecha creación: 2026-08-19  
Estado: producción  
Tipo: vista (panel exclusivo CEO/Desarrollador)

---

## ¿Qué hace? (para el usuario)

Panel de administración para crear y activar/desactivar las comunidades de Product HUB, exclusivo para roles `CEO` y `Desarrollador`. Es un módulo **independiente**, deliberadamente desacoplado del módulo de usuario final `dropshipping/product-hub` (ver ADR). Muestra:

- Tabla paginada de todas las comunidades (activas e inactivas), con búsqueda por nombre de comunidad/tienda dueña y filtro de estado.
- Botón "Nueva comunidad": elige la tienda dueña (solo tipo `Proveedor`), nombre, descripción y administradores. Opcionalmente crea también una comunidad social vinculada en el muro compartido.
- Edición de nombre/descripción/administradores de una comunidad existente (la tienda dueña no es editable una vez creada).
- Activar/desactivar una comunidad, con confirmación previa.

---

## Ruta

```
administracion/administrar-comunidades
```

---

## Decoradores y configuración técnica

```typescript
@Component({
  selector: 'app-administrar-comunidades',
  imports: [CommonModule, ComunidadFormModalComponent, CardComponent, CardBodyComponent],
  templateUrl: './administrar-comunidades.component.html',
  styleUrl: './administrar-comunidades.component.scss',
})
export class AdministrarComunidadesComponent implements OnInit
```

**Ubicación:** `src/app/views/administracion/administrar-comunidades/administrar-comunidades.component.ts`

---

## Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `tieneAcceso` | `boolean` | Resuelto en el constructor vía `repo.puedeCrearComunidad()` — bloquea toda la vista si es `false` |
| `comunidadesFiltradas` | `Signal<Comunidad[]>` | Aplica búsqueda + filtro de estado sobre todas las comunidades |
| `comunidadesPaginadas` | `Signal<Comunidad[]>` | Slice de `comunidadesFiltradas` según `currentPage`/`pageSize` |
| `formMode` | `Signal<FormMode>` | `'nueva'` o `'edicion'` |
| `form` | `FormGroup` | `idTiendaDuena`, `nombreComunidad`, `descripcionComunidad`, `adminsComunidad`, `crearComunidadSocial` |

---

## Servicios y endpoints

| Método (`AdministrarComunidadesRepository`) | Colección | Descripción |
|---|---|---|
| `puedeCrearComunidad()` | — | Rol `CEO`/`Desarrollador` vía `sessionStorage` |
| `getUsuarios()` | `Users` | Selector de administradores |
| `getTiendas()` | `Tienda` | Se filtra en el componente a `TipoTienda` incluye `'Proveedor'` |
| `getTodasLasComunidades()` | `ProductHubComunidades` | Activas e inactivas, sin filtrar |
| `crearComunidad(...)` | `ProductHubComunidades` | Inserta con `IdComunidad` nuevo (UUID v4) |
| `crearComunidadSocial(...)` | `Comunidades` | Solo si el checkbox del form está marcado |
| `editarComunidad(id, ...)` | `ProductHubComunidades` | Nombre/descripción/admins — no toca tienda dueña |
| `toggleEstadoComunidad(c, activar)` | `ProductHubComunidades` | Activa/desactiva |

Repository propio e independiente de `ProductHubRepository` — ver [ADR-002](../../../arquitectura/decisiones.md) para la razón del desacople.

---

## Flujo principal

```
ngOnInit()
  └─► !tieneAcceso → alerta "Sin permiso", no carga nada
  └─► tieneAcceso  → Promise.all
        ├─► repo.getUsuarios()
        ├─► repo.getTiendas()      → filtra tipo Proveedor
        └─► repo.getTodasLasComunidades()

Usuario crea comunidad
  └─► guardar() [formMode 'nueva']
        ├─► crearComunidadSocial marcado → repo.crearComunidadSocial(...)
        └─► repo.crearComunidad(idTiendaDuena, ..., adminsComunidad)
        └─► refetch completo (el insert no devuelve el _id de Mongo)

Usuario edita comunidad
  └─► guardar() [formMode 'edicion']
        └─► repo.editarComunidad(id, ...) → merge local, sin refetch

Usuario activa/desactiva
  └─► toggleEstadoComunidad(c) → confirmación Swal → repo.toggleEstadoComunidad()
```

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-08-19 | Adalberto González | Documentación inicial del módulo, ya en producción |

---

## Observaciones

- El `IdComunidad` se genera con `crearIdComunidad()` (UUID v4) en vez de un consecutivo, para evitar condición de carrera entre creaciones simultáneas y no exponer IDs predecibles/enumerables.
- Comparte la colección física `ProductHubComunidades` con el módulo de usuario final, pero no comparte código: modelos y repository están deliberadamente duplicados (~40 líneas), documentado en [ADR-002](../../../arquitectura/decisiones.md).
