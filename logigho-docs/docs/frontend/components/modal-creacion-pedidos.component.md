---

## Autor: Adalberto González

Fecha creación: 2026-08-19  
Estado: producción  
Tipo: componente

# Componente: ModalCreacionPedidosComponent

**Selector:** `app-modal-creacion-pedidos` 
**Ubicación:** `src/app/components/modal-creacion-pedidos/modal-creacion-pedidos.component.ts`  
**Acceso:** Autenticado | usado desde las vistas de creación de pedidos

---

## ¿Qué hace?

Modal de 4 pasos para crear un pedido manual: elegir productos del catálogo (o declarar valor sin productos), completar datos de envío, cotizar transportadora y generar la guía. Es un componente grande y preexistente — esta doc no cubre cada paso en detalle, solo su estructura general y el punto donde se integra con Product HUB.

---

## Decoradores

| Decorador | Nombre | Tipo | Descripción |
|---|---|---|---|
| `@Input` | `isOpen` | `boolean` | Abre el modal y dispara `resetModalState()` al pasar a `true` |
| `@Output` | `closeEvent` | `EventEmitter<void>` | Emitido al cerrar el modal |
| `@Output` | `refreshDataEvent` | `EventEmitter<void>` | Emitido para que el padre recargue su listado |

---

## Pasos del modal

1. **Productos** — catálogo paginado (24 por página) con búsqueda server-side, o modo "valor declarado" sin productos concretos.
2. **Envío** — ciudad origen/destino, dimensiones, tienda.
3. **Cotización** — cotiza contra Interrapidísimo / Servientrega / Envía.
4. **Guía** — datos del destinatario y generación final.

---

## Visibilidad de productos en el catálogo (paso 1)

El catálogo no muestra todos los productos del sistema — filtra según quién está viendo. La regla, aplicada tanto en memoria (`esProductoVisibleParaUsuario`) como en el pipeline Mongo que trae la página (`construirPipelineProductos`):

1. Usuario con tienda `"Todas"` → ve el catálogo completo, sin filtro.
2. Producto `perfilproducto: 'publico'` → visible para cualquiera.
3. Producto `perfilproducto: 'privado'` de una tienda asignada al usuario → visible.
4. **Producto `'privado'` de OTRA tienda, con autorización vigente de Product HUB** → visible. Este caso es la integración agregada: `resolverAutorizacionesProductHub()` resuelve, una sola vez al abrir el modal, el conjunto de `idproducto` con `ProductHubAutorizaciones.Estado:true` para alguna tienda del usuario, y lo cachea en `idsProductoAutorizadosProductHub`. Ese conjunto se suma como una rama `$or` más en el pipeline y como condición extra en el filtro en memoria — nunca resta lo que ya era visible por las reglas 1-3.

Si el usuario tiene `"Todas"`, no se hace ningún fetch a `ProductHubAutorizaciones` (regla 1 ya cubre todo). Si la resolución falla, el catálogo sigue funcionando con el comportamiento previo a Product HUB (reglas 1-3), sin bloquear el modal.

---

## Servicios y endpoints

| Servicio | Método | Endpoint | Cuándo |
|---|---|---|---|
| `ConsumoGenericoService` | `consultarConPipeline` | `Productos` (pipeline con `$match`/`$skip`/`$limit`) | Carga y paginación del catálogo |
| `ConsumoGenericoService` | `consultarGenerico` | `metodoGenerico?coleccion=Tienda` | Resolver ids de tiendas asignadas (Product HUB) |
| `ConsumoGenericoService` | `consultarGenerico` | `metodoGenerico?coleccion=ProductHubAutorizaciones` | Resolver autorizaciones vigentes del usuario |
| `ConsumoGenericoService` | `consultarGenerico` | `metodoGenerico?coleccion=Ciudades` | Datalist de ciudades |
| Servicios de cotización | — | Interrapidísimo / Servientrega / Envía | Paso 3 |

---

## Flujo principal (integración Product HUB)

```
fetchTableData()  [carga inicial del catálogo]
  -> resolverAutorizacionesProductHub()
       -> si usuarioTieneTodasLasTiendas() -> Set vacío, sin fetch
       -> si no:
            -> resuelve Id numérico de mis tiendas asignadas (coleccion=Tienda)
            -> GET ProductHubAutorizaciones (Estado:true, IdTiendaSolicitante en mis tiendas)
            -> idsProductoAutorizadosProductHub = Set<idproducto>
  -> cargarProductos(1, '', false)
       -> construirPipelineProductos() incluye idsProductoAutorizadosProductHub en el $or

Búsqueda con debounce / paginación posterior
  -> reusa el mismo Set ya cacheado, no vuelve a consultar ProductHubAutorizaciones

Validación de IDs pegados a mano (modo valor-declarado)
  -> esProductoVisibleParaUsuario() aplica el mismo Set en memoria
```

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-08-19 | Adalberto González | Integración con Product HUB: productos privados con autorización vigente amplían el catálogo visible, sin afectar las reglas de público/privado-propio ya existentes |

---

## Observaciones

- `idsProductoAutorizadosProductHub` se resuelve una sola vez por apertura del modal y se reutiliza en cada página/búsqueda — no hay una consulta nueva por cada scroll.
- Un `IdStockN` guardado como string en algún documento legacy de `PedidosInter` no afecta a este componente (es un problema del lado de `gestion-comunidad` → Estadísticas, no de creación de pedidos).
- Ver `frontend/views/dropshipping/product-hub/product-hub-flujo.md` para el flujo de negocio completo de cómo un producto llega a tener una autorización vigente.
