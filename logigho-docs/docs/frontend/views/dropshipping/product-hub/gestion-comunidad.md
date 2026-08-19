# Módulo: Product HUB — Gestión de Comunidad

---

## Autor: Adalberto González
Fecha creación: 2026-08-19  
Estado: producción  
Tipo: vista (panel privado de administradores)

---

## ¿Qué hace? (para el usuario)

Panel privado accesible solo para los `AdminsComunidad` de una comunidad. Tiene un sidebar vertical con 5 secciones:

- **Miembros**: tabla paginada de quién pertenece a la comunidad (usuario + tienda con la que se unió), con opción de expulsar.
- **Solicitudes de membresía**: bandeja de tiendas que piden unirse, con aprobar/rechazar y motivo obligatorio.
- **Solicitudes de producto**: bandeja de miembros que piden permiso para vender un producto privado específico.
- **Estadísticas**: dos rankings de ventas reales (tiendas autorizadas que más vendieron, productos más vendidos), con selector de métrica (cantidad/monto/pedidos) y tipo de gráfico (barras/línea/área polar) vía CoreUI Chart.js.
- **Editar**: nombre, descripción y administradores de la comunidad.

---

## Ruta

```
dropshipping/product-hub/comunidad/:id/gestion
```

---

## Decoradores y configuración técnica

```typescript
@Component({
  selector: 'app-gestion-comunidad',
  imports: [
    CommonModule, ReactiveFormsModule, RouterLink, CardComponent, CardBodyComponent,
    RowComponent, ColComponent, BadgeComponent, ChartjsComponent, SolicitudModalComponent,
  ],
  templateUrl: './gestion-comunidad.component.html',
  styleUrl: './gestion-comunidad.component.scss',
})
export class GestionComunidadComponent implements OnInit
```

**Ubicación:** `src/app/views/dropshipping/product-hub/gestion-comunidad/gestion-comunidad.component.ts`

---

## Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `tabActiva` | `Signal<TabGestion>` | `'miembros' \| 'solicitudes-membresia' \| 'solicitudes-producto' \| 'estadisticas' \| 'editar'` |
| `solicitudesMembresiaPendientes` / `solicitudesProductoPendientes` | `Signal<[]>` | Bandejas filtradas en cliente por `IdComunidad` + `Estado:false` |
| `estadisticas` | `Signal<EstadisticasVentasComunidad \| null>` | Se carga perezosamente, solo la primera vez que se visita el tab |
| `metricaEstadisticas` / `rankingEstadisticas` / `tipoGraficoEstadisticas` | `Signal` | Controles del gráfico: métrica, ranking activo y tipo de chart |
| `modalAbierto` | `Signal<ModalAbierto \| null>` | Solicitud (membresía o producto) actualmente abierta en el modal de resolución |
| `formEditar` | `FormGroup` | `nombreComunidad`, `descripcionComunidad`, `adminsComunidad` |

---

## Servicios y endpoints

| Método (`ProductHubRepository`) | Qué hace |
|---|---|
| `esAdminDeComunidad(c)` | Gate de acceso al cargar la vista |
| `getSolicitudesMembresia()` / `getSolicitudesProducto()` | Traen todo y se filtra en cliente por comunidad + pendientes |
| `getUsuariosMiembros(idComunidad, emails)` | Resuelve usuario + tienda de cada miembro |
| `expulsarMiembro(comunidad, email)` | Quita del array `MiembrosComunidad` |
| `aprobarSolicitudMembresia` / `rechazarSolicitudMembresia` | Ver detalle abajo |
| `aprobarSolicitudProducto` / `rechazarSolicitudProducto` | Ver detalle abajo |
| `getEstadisticasVentasComunidad(idTiendaDuena)` | Ver detalle abajo |
| `getUsuarios()` | Selector de administradores en el tab Editar |

### `aprobarSolicitudMembresia` — doble escritura sin transacciones

Primero marca la solicitud como `Resultado:'APROBADO'`, y **solo si eso tuvo éxito** agrega el email a `MiembrosComunidad` de la comunidad. No hay transacción entre ambos `PUT` — si el segundo paso fallara, la solicitud quedaría aprobada sin membresía efectiva, pero eso no ha ocurrido en la práctica.

### `aprobarSolicitudProducto` — igual patrón, con manejo explícito de fallo parcial

Aprueba la solicitud y luego inserta una `Autorizacion` (`Estado:true`). Si el insert falla, el método retorna `{ autorizacionActivada: false }` y la vista muestra una alerta pidiendo activarla manualmente — la solicitud queda aprobada de todos modos. La autorización nunca se borra; solo se podría revocar cambiando `Estado` a `false`, pero hoy no existe UI para esa acción.

### `getEstadisticasVentasComunidad` — pipeline de agregación sobre `PedidosInter`

Cruza los `idproducto` de la tienda dueña (traídos de `Productos`) contra `PedidosInter`, filtrando solo los estados homologados a "venta real": `Digitalizada`, `Entregada`, `Archivada`, `Pagada`, `Facturado` — ningún otro estado cuenta. Como los campos de stock en `PedidosInter` son planos (`IdStock1..12`, `CantidadStock1..12`, `PrecioPorUnidadStock1..12`), el pipeline los proyecta a un array de 12 slots y los aplana con `$unwind` antes de agrupar. Usa un pre-`$match` con `$or` sobre los 12 campos para descartar la mayoría de documentos antes del `$unwind` — sin eso, el aggregate excedía el timeout del servidor. Envía dos pipelines en paralelo (agrupado por tienda y por producto) en vez de un solo `$facet`, porque el backend de este proyecto no soporta esa etapa. Codifica el pipeline en base64, mismo patrón que `relacion-despacho`.

---

## Flujo principal

```
ngOnInit()
  └─► getComunidad(id)
        └─► si no soy admin → alerta + redirect a comunidad-detalle
  └─► formEditar.patchValue(...)
  └─► repo.getUsuarios()
  └─► recargarBandejas() + recargarMiembros()

setTab('estadisticas')  [primera vez]
  └─► cargarEstadisticas() → repo.getEstadisticasVentasComunidad(IdTiendaDuena)

Admin resuelve una solicitud (modal)
  └─► resolverModal(ev)
        ├─► membresia + APROBAR → repo.aprobarSolicitudMembresia() → agrega a miembros local
        ├─► membresia + RECHAZAR → repo.rechazarSolicitudMembresia()
        ├─► producto + APROBAR  → repo.aprobarSolicitudProducto() → alerta si autorizacionActivada:false
        └─► producto + RECHAZAR → repo.rechazarSolicitudProducto()

Admin guarda el tab Editar
  └─► guardarEdicion() → PUT sobre ProductHubComunidades (Nombre/Descripcion/Admins)
```

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-08-19 | Adalberto González | Documentación inicial del módulo, ya en producción |

---

## Observaciones

- `c-chart` no soporta cambiar `[type]` sobre una instancia ya creada — el HTML repite tres `<c-chart>` con `*ngIf` (uno por tipo) en vez de uno solo con el tipo como binding.
- Las tablas de miembros y administradores usan paginación en cliente compartida (`POR_PAGINA_LISTA = 10`).
