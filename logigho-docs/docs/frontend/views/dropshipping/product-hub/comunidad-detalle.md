# Módulo: Product HUB — Detalle de Comunidad

---

## Autor: Adalberto González
Fecha creación: 2026-08-19  
Estado: producción  
Tipo: vista

---

## ¿Qué hace? (para el usuario)

Es la pantalla de un miembro dentro de una comunidad ya aprobada. Solo es accesible para quien ya pertenece (dueño, admin o miembro) — si no, se muestra una alerta y se redirige de vuelta al explorador. Tiene dos pestañas:

- **Publicaciones**: el muro social de la comunidad (si sus admins también moderan una comunidad social del muro compartido). Permite reaccionar (like) y comentar, sujeto a permisos del post y a que el usuario no esté restringido en el muro.
- **Productos**: el catálogo completo de la tienda dueña, agrupado por producto con sus variaciones, con buscador y chips de categoría. Cada producto muestra si el usuario ya tiene autorización para venderlo, tiene una solicitud pendiente, o puede solicitar permiso.
- Un menú de "3 puntos" en el banner ofrece "Gestionar comunidad" (solo admins) y "Salir de la comunidad" (solo miembros no-admin).

---

## Ruta

```
dropshipping/product-hub/comunidad/:id
```

---

## Decoradores y configuración técnica

```typescript
@Component({
  selector: 'app-comunidad-detalle',
  imports: [
    CommonModule, RouterLink, CardComponent, CardBodyComponent, RowComponent, ColComponent,
    SolicitudModalComponent, SeleccionTiendaModalComponent, MenuAccionesComponent,
  ],
  templateUrl: './comunidad-detalle.component.html',
  styleUrl: './comunidad-detalle.component.scss',
})
export class ComunidadDetalleComponent implements OnInit
```

**Ubicación:** `src/app/views/dropshipping/product-hub/comunidad-detalle/comunidad-detalle.component.ts`

---

## Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `esMiembro` / `esAdmin` | `Signal<boolean>` | Gate de acceso y de opciones del menú |
| `opcionesMenu` | `Signal<OpcionMenu[]>` | Opciones del menú de 3 puntos, según `esAdmin`/`puedeSalirDeComunidad` |
| `productosVisibles` / `productosFiltrados` | `Signal<ProductoAgrupado[]>` | Catálogo paginado en cliente (12 por página) con búsqueda y filtro de categoría |
| `autorizacionesVigentes` / `solicitudesProductoPendientes` | `Signal<Set<number>>` | `idproducto` con permiso activo o solicitud en curso, para pintar el estado en cada card |
| `puedeInteractuarEnMuro` | `Signal<boolean>` | `true` si `Users.EstadoComentario === 'ACTIVO'` y no hay restricción vigente |
| `publicaciones` | `Signal<PublicacionSocial[]>` | Feed paginado (5 por página) de la comunidad social vinculada por admins compartidos |

---

## Servicios y endpoints

| Método (`ProductHubRepository`) | Colección | Uso |
|---|---|---|
| `getComunidadSocialVinculada(adminsComunidad)` | `Comunidades` | Resuelve el muro vinculado por coincidencia de admin/moderador |
| `getPublicacionesDeComunidadSocial(admins)` | `Publicaciones` | Feed del tab Publicaciones |
| `puedeInteractuarEnMuro()` | `Users`, `UsuariosRestringidosMuro` | Gate de reacción/comentario |
| `getUsuariosMiembros(idComunidad, emails)` | `ProductHubSolicitudesMembresia` | Usuario + tienda de cada miembro, para el widget de miembros |
| `getTiendaFijadaEnComunidad(idComunidad, email)` | `ProductHubSolicitudesMembresia` | Tienda con la que ya soy miembro — evita repreguntar al solicitar producto |
| `getAutorizacionesVigentes` / `getSolicitudesProductoPendientes` | `ProductHubAutorizaciones`, `ProductHubSolicitudesProducto` | Estado de permiso por producto |
| `crearSolicitudProducto(params)` | `ProductHubSolicitudesProducto` | Envía la solicitud de venta de un producto privado |
| `expulsarMiembro(comunidad, email)` | `ProductHubComunidades` | Reutilizado aquí para la salida voluntaria |

La vista también consulta `Productos` (catálogo de la tienda dueña) y `Publicaciones`/reacciones/comentarios directamente vía `ConsumoGenericoService`.

---

## Flujo principal

```
ngOnInit()
  └─► getComunidad(id)
        ├─► no existe → mensaje de error
        └─► existe, pero no soy miembro → alerta + redirect a explorar-comunidades
  └─► si soy miembro, Promise.all
        ├─► getProductosDeTienda(IdTiendaDuena)
        ├─► repo.getComunidadSocialVinculada(admins)
        ├─► repo.puedeInteractuarEnMuro()
        ├─► getMiTiendaSilenciosa()
        ├─► repo.getUsuariosMiembros(...)
        └─► repo.getTiendaFijadaEnComunidad(...)
  └─► si hay muro vinculado → cargarPublicaciones()
  └─► si se resolvió mi tienda → cargarEstadoAutorizaciones()

Usuario hace clic en "Solicitar permiso" sobre un producto
  └─► abrirSolicitudProducto(p)
        ├─► ya tengo tienda fijada en esta comunidad → abre modal directo
        ├─► sin tienda fijada, 1 elegible  → abre modal directo
        └─► sin tienda fijada, 2+ elegibles → selector de tienda primero
  └─► enviarSolicitudProducto() → repo.crearSolicitudProducto()

Menú "3 puntos" → onAccionMenu(id)
  ├─► 'gestionar' → navega a gestion-comunidad
  └─► 'salir'     → confirmación + repo.expulsarMiembro(comunidad, miEmail)
```

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-08-19 | Adalberto González | Documentación inicial del módulo, ya en producción |

---

## Observaciones

- Los comentarios del muro son planos (`ComentarioSocial[]`), sin hilos anidados — `comentarioPadreId` existe en el modelo pero no se usa todavía en esta vista.
- La reacción es un "like" simple (toggle), no el selector completo de 6 emociones que existe en el módulo de red social.
- `esProductoDeMiTienda()` oculta el botón de solicitud cuando la tienda del usuario es la misma tienda dueña de la comunidad — no tendría sentido pedirse permiso a sí misma.
