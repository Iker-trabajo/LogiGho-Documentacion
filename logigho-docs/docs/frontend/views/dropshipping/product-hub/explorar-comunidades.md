# Módulo: Product HUB — Explorar Comunidades

---

## Autor: Adalberto González
Fecha creación: 2026-08-19  
Estado: producción  
Tipo: vista (landing del módulo)

---

## ¿Qué hace? (para el usuario)

Es la puerta de entrada al Product HUB: el marketplace B2B interno donde una tienda ("dueña") comparte su catálogo con otras tiendas mediante comunidades. Al abrir la pantalla se muestran:

- Un **feed central** con las comunidades ajenas activas (a las que el usuario no pertenece), paginado con botón "Cargar más" y buscador por nombre de comunidad o tienda dueña.
- Un **widget "Mis comunidades"** en el sidebar, con las comunidades donde el usuario ya es admin, dueño o miembro.
- Un **widget "Comunidades destacadas"**, ordenado por el stock total del catálogo de cada tienda dueña (proxy de actividad).
- Un botón para **solicitar unirse** a una comunidad ajena. Si el usuario tiene varias tiendas elegibles, primero se le pide elegir a nombre de cuál actúa.
- El botón cambia según el estado de la solicitud: sin solicitar / pendiente / rechazada (puede volver a solicitar) — nunca aparece si ya es miembro.

---

## Ruta

```
dropshipping/product-hub
```

---

## Decoradores y configuración técnica

```typescript
@Component({
  selector: 'app-explorar-comunidades',
  imports: [
    CommonModule, RowComponent, ColComponent, CardComponent, CardBodyComponent,
    ButtonDirective, SolicitudModalComponent, SeleccionTiendaModalComponent,
  ],
  templateUrl: './explorar-comunidades.component.html',
  styleUrl: './explorar-comunidades.component.scss',
})
export class ExplorarComunidadesComponent implements OnInit
```

**Ubicación:** `src/app/views/dropshipping/product-hub/explorar-comunidades/explorar-comunidades.component.ts`

---

## Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `misComunidades` / `misComunidadesVisibles` | `Signal<Comunidad[]>` | Comunidades donde soy admin, tienda dueña o miembro aprobado |
| `comunidadesAjenas` / `comunidadesAjenasVisibles` | `Signal<Comunidad[]>` | Feed principal, filtrado por búsqueda y paginado en cliente (12 por página) |
| `comunidadesDestacadas` | `Signal<Comunidad[]>` | Top 5 comunidades ajenas por stock total del catálogo de la tienda dueña |
| `estadosSolicitud` | `Signal<Map<string, SolicitudMembresia>>` | Estado de mi solicitud por `IdComunidad`, agregado entre todas mis tiendas |
| `comunidadSolicitando` | `Signal<Comunidad \| null>` | Comunidad con el modal de solicitud abierto |
| `tiendasParaElegir` | `Signal<{id,nombre}[]>` | Alimenta el modal de selección de tienda cuando hay más de una elegible |

---

## Servicios y endpoints

| Método (`ProductHubRepository`) | Colección | Uso en esta vista |
|---|---|---|
| `getTiendasElegibles()` | `Tienda` | Mis tiendas `Propia`/`Dropshipping`, para resolver a nombre de cuál se solicita |
| `getEstadosMembresiaAgregados(idsComunidad, idsTiendas, email)` | `ProductHubSolicitudesMembresia` | Estado agregado de solicitud por comunidad (ver nota abajo) |
| `crearSolicitudMembresia(params)` | `ProductHubSolicitudesMembresia` | Envía la solicitud de unión |

La vista también consulta directamente `ProductHubComunidades` (comunidades activas) y `Productos` (stock por tienda, para el ranking de destacadas), sin pasar por el repository.

**Patrón de URL:** `metodoGenerico?coleccion=<COLECCION>&mcomp=2`

---

## Flujo principal

```
ngOnInit()
  └─► Promise.all
        ├─► repo.getTiendasElegibles()
        ├─► getComunidadesActivas()   // Estado:true
        └─► getStockPorTienda()       // para destacadas
  └─► cargarEstadosSolicitud() → repo.getEstadosMembresiaAgregados(...)

Usuario hace clic en "Solicitar unirme"
  └─► abrirSolicitud(c)
        ├─► 0 tiendas elegibles → alerta, no continúa
        ├─► 1 tienda elegible   → abre directo el modal de solicitud
        └─► 2+ tiendas          → abre selector de tienda primero
              └─► elegirTienda() → abre el modal de solicitud

Usuario envía el motivo en SolicitudModalComponent
  └─► enviarSolicitud()
        └─► repo.crearSolicitudMembresia() → actualiza estadosSolicitud en memoria (sin refetch)
```

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-08-19 | Adalberto González | Documentación inicial del módulo, ya en producción |

---

## Observaciones

- `getEstadosMembresiaAgregados` filtra por `IdTiendaSolicitante` **y** por `EmailSolicitante`. Filtrar solo por tienda es un bug ya corregido: para usuarios con acceso a "Todas las tiendas", `idsTiendas` cubre todas las tiendas elegibles del sistema, así que una solicitud aprobada de OTRO usuario se atribuía por error a cualquier usuario "Todas".
- La tienda con la que un usuario se une a una comunidad queda fija para siempre en esa comunidad — no se vuelve a preguntar en `comunidad-detalle` ni al pedir autorización de producto.
