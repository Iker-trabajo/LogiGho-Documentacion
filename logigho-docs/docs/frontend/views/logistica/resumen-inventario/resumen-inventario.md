# Módulo: Resumen de Inventario

---

## Autor: Iker Acevedo
Fecha creación: 2026-05-26
Estado: desarrollo
Tipo: módulo (1 vista + 7 componentes hijos)

---

## Índice

1. [Vista: ResumenInventarioComponent](#1-vista-resumeninventariocomponent)
2. [Componente: KpiCardComponent](#2-componente-kpicardcomponent)
3. [Componente: BajoStockModalComponent](#3-componente-bajostockmodalcomponent)
4. [Componente: ProductosResumenTablaComponent](#4-componente-productosresumentablacomponent)
5. [Componente: VistaProveedoresComponent](#5-componente-vistaproveedorescomponent)
6. [Componente: FiltrosInventarioComponent](#6-componente-filtrosinventariocomponent)
7. [Componente: KpiDetalleModalComponent](#7-componente-kpidetalleModalcomponent)
8. [Componente: ResumenModalComponent](#8-componente-resumenmodalcomponent)

---

## 1. Vista: ResumenInventarioComponent

**Selector:** `app-resumen-inventario`
**Ubicación:** `src/app/views/logistica/resumen-inventario/resumen-inventario.component.ts`
**Acceso:** Logística → Resumen Inventario

---

### ¿Qué hace? (para el usuario)

Es la pantalla principal del resumen de inventario. Al abrirla, carga automáticamente todos los datos de los productos desde el sistema y muestra:

- **4 tarjetas KPI** en la parte superior con el stock total, productos activos, unidades en tránsito y productos en bajo stock.
- **Filtros** para seleccionar tiendas por ecosistema y para filtrar por estado del producto (Activo / Inactivo / Todos).
- **Una tabla detallada** con todos los productos, sus movimientos y costos de inventario.
- **Una vista por proveedor** con tarjetas que agrupan el inventario por cada proveedor.
- El operario puede hacer clic en la tarjeta de Bajo Stock para ver exactamente qué productos están en alerta.
- Puede exportar los datos a Excel.
- Puede ajustar el umbral global de bajo stock desde el header.

---

### Ruta

```
logistica/resumen-inventario
```

---

### Decoradores y configuración técnica

```typescript
@Component({
  selector: 'app-resumen-inventario',
  standalone: true,
  imports: [
    CommonModule, FormsModule,
    KpiCardComponent, BajoStockModalComponent,
    ProductosResumenTablaComponent, VistaProveedoresComponent,
    FiltrosInventarioComponent, KpiDetalleModalComponent,
    ResumenModalComponent
  ],
  templateUrl: './resumen-inventario.component.html',
  styleUrl: './resumen-inventario.component.scss',
})
export class ResumenInventarioComponent implements OnInit
```

Implementa `OnInit` para disparar la carga de datos al inicializar la vista.

---

### Interfaz central del módulo

Toda la información del inventario en memoria sigue este contrato, derivado de la colección `ResumenInventario`:

```typescript
export interface ResumenItem {
  IdProducto: string;     // ID único del producto
  Nombre: string;         // Nombre comercial
  Tienda: string;         // Tienda/proveedor asignado
  Variacion: string;      // Variación del producto (ej: "TALLA XL")
  Color: string;          // Color del producto
  Ingresos: number;       // Unidades que ingresaron
  Salidas: number;        // Unidades que salieron
  Devoluciones: number;   // Unidades devueltas
  Anulaciones: number;    // Unidades anuladas
  Ajustes: number;        // Ajustes manuales
  Entregas: number;       // Unidades entregadas
  Actual: number;         // Stock actual en bodega
  Consumo: number;        // Consumo registrado
  EnTransito: number;     // Unidades en tránsito (puede ser negativo)
  PrecioProveedor: string;// Costo unitario del proveedor (string de número)
  fecha: string;          // Fecha del último cálculo (ISO 8601)
}
```

---

### Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `isLoading` | `boolean` | Activa el spinner mientras se cargan los datos |
| `errorCarga` | `boolean` | Bandera de error si el backend falla |
| `datos` | `ResumenItem[]` | Todos los registros del backend sin filtrar |
| `umbralBajoStock` | `number` | Umbral global por defecto (1000). El usuario puede cambiarlo |
| `tiendaMap` | `Map<string, string[]>` | Ecosistema → lista de tiendas. Alimenta el filtro |
| `tiendasSeleccionadas` | `Set<string>` | Tiendas actualmente marcadas en el filtro |
| `tiendasProveedor` | `Set<string>` | Tiendas cuyo único TipoTienda es "Proveedor" |
| `estadoProductoMap` | `Map<string, string>` | IdProducto → "Activo" o "Inactivo" |
| `stockMinimoMap` | `Map<string, {_id, StockMinimo}>` | Stock mínimo configurado por producto |
| `imagenProductoMap` | `Map<string, string>` | IdProducto → URL de imagen |
| `filtroEstado` | `string` | Estado seleccionado en UI ("Activo" por defecto) |
| `modalBajoStockAbierto` | `boolean` | Controla visibilidad del modal de bajo stock |
| `modalKpiAbierto` | `boolean` | Controla visibilidad del modal de detalle KPI |
| `modalResumenAbierto` | `boolean` | Controla visibilidad del modal resumen general |

**Getters computados (valores que Angular recalcula automáticamente):**

| Getter | Retorna | Lógica |
|---|---|---|
| `datosVista` | `ResumenItem[]` | Aplica filtros de tiendas + estado sobre `datos` |
| `stockTotal` | `number` | Suma de `Actual` de todos los productos en `datosVista` |
| `productosActivos` | `number` | Cantidad de productos con `Actual > 0` |
| `enTransito` | `number` | Suma de `EnTransito` de todos los productos |
| `productosBajoStock` | `ResumenItem[]` | Productos donde `Actual <= mínimo` (personalizado o global) |
| `productosActivosList` | `ResumenItem[]` | Lista de productos con `Actual > 0` |

---

### Servicios y endpoints

| Servicio | Uso |
|---|---|
| `ConsumoGenericoService` | Consultas y mutaciones al backend |
| `DecompressionService` | Descomprime respuestas Zstd del backend |

**Patrón de URL:**
```
metodoGenerico?coleccion=<COLECCION>&Tienda=<TIENDAS>&mcomp=2
```
> Si el usuario tiene acceso a "Todas" las tiendas, se omite `&Tienda=` para que el backend devuelva todos los registros.

**Endpoints por colección:**

| Colección | Propósito |
|---|---|
| `ResumenInventario` | Datos principales del inventario (stock, movimientos) |
| `Tienda` | Ecosistemas y tipos de tienda |
| `Productos` | Estado (Activo/Inactivo) e imagen de cada producto |
| `StockMinimoProducto` | Mínimos configurados por producto |

**Mutaciones (guardarStockMinimo):**
- **PUT** `metodoGenerico?coleccion=StockMinimoProducto` si el producto ya tiene configuración
- **POST** `metodoGenerico?coleccion=StockMinimoProducto` si es la primera vez que se configura

---

### Flujo principal

```
ngOnInit()
  └─► cargarDatos() [Promise.all en paralelo]
        ├─► cargarResumen()        → datos[]
        ├─► cargarMapaTiendas()    → tiendaMap + tiendasProveedor
        ├─► cargarProductos()      → estadoProductoMap + imagenProductoMap
        └─► cargarStockMinimo()    → stockMinimoMap

Usuario interactúa con filtros
  └─► onFiltroChange() / onEstadoChange()
        └─► datosVista (getter) recalcula en cada CD

Usuario hace clic en card "Bajo Stock"
  └─► abrirKpiDetalle() → modalKpiAbierto = true

Usuario edita stock mínimo en la tabla
  └─► guardarStockMinimo() [upsert]
        ├─► existente → PUT
        └─► nuevo     → POST + recarga colección
```

---

### Historial de cambios

| Fecha | Cambio |
|---|---|
| 2026-05-26 | Creación del módulo con carga paralela de 4 colecciones |
| 2026-05-26 | Agregado sistema de stock mínimo por producto con upsert |
| 2026-05-26 | Refactor html2canvas removido, modal compactado para mayor densidad de datos |

---

### Observaciones

- `getTiendaParam()` lee `sessionStorage.getItem('tiendas_asignadas')`. Si el array incluye `"Todas"`, la URL no lleva filtro de tienda.
- `guardarStockMinimo()` crea `new Map(this.stockMinimoMap)` después de mutar el mapa para forzar la detección de cambios de Angular (los Maps mutables no disparan CD por sí solos).
- El `Promise.all()` en `cargarDatos()` hace las 4 peticiones en paralelo, reduciendo el tiempo de carga total.
