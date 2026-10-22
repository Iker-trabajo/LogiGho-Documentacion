---

## Autor: Iker Acevedo  
Fecha creacion: 2026-10-10  
Estado: aceptada

# ADR-INV-004 — Filtro de activación por proveedor

**Autor:** Iker Acevedo  
**Fecha:** 2026-10-09 (INV-003) · 2026-10-10 (INV-034)  
**Estado:** Aceptada

---

## Contexto

El inventario es un servicio que se vende a ciertas tiendas. En VAL-1 (hallazgos H-1 y H-2), con el filtro apagado el sistema reescribía productos inactivos y creaba salidas de pedidos anteriores al primer ingreso del producto. Solo el ecosistema BTK (65 productos activos) usa inventario real.

## Opciones consideradas

### Opción A — Procesar todo producto que exista en `Productos`

**Pros:** sin configuración.  
**Contras:** reescribe la historia de productos que nadie gestiona.

### Opción B — Activar por banderas en el proveedor (dueño del producto)

**Pros:** solo se sincroniza lo contratado; no se crea stock fantasma.  
**Contras:** los productos nuevos no descuentan hasta activarse.

---

## Decisión

**Se eligió:** Opción B, con `FILTRO_ACTIVACION=on` en el go-live.

**Razón:** un producto cuenta solo si `Productos.ManejaInventario=true`, la tienda dueña (`Productos.IdTienda = Tienda.Id`) tiene `ServicioInventario=true` y la fecha del pedido es ≥ `FechaInicioInventario` (el primer ingreso). La activación vive en el **proveedor**; la tienda que vende no necesita nada. Un producto sin ingreso no se activa. Los ingresos y ajustes se escriben siempre.

---

## Consecuencias

**Positivas:** no se toca lo que nadie gestiona; los inactivos no se limpian ni se recalculan.  
**Negativas:** no hay una pantalla completa de activación para todos los casos (*Administrar inventario* cubre a Jefe Logistica, Desarrollador y CEO, y también hay scripts); hay que avisar al Jefe de Logística que los demás productos y los nuevos dejan de sincronizar.

---

## Impacto en el código

| Módulo / Repo | Cambio |
|---|---|
| `ApiLambdaInventario` | `ReglaActivacion`, `ProductoRepository` |
| Base de datos | Campos `Productos.ManejaInventario`, `Productos.FechaInicioInventario`, `Tienda.ServicioInventario` (script `sembrar_campos_activacion_prod.py`) |
| `SitioLogiGho` | Pantalla *Administrar inventario* |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-10-21 | Iker Acevedo | Documentado en MkDocs |

---

## Referencias

- [Arquitectura — filtro de activación](../../../backend/inventario/arquitectura.md#filtro-de-activacion-por-proveedor)
