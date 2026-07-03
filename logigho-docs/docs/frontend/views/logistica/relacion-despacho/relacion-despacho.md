# Módulo: Relación de Despacho

---

## Autor: Adalberto González
Fecha creación: 2026-07-03  
Estado: desarrollo  
Tipo: módulo (1 vista + 5 componentes hijos + 1 servicio PDF)

---

## Índice

1. [Vista: RelacionDespachoComponent](./relacion-despacho.md)
2. [Componente: PistoleroComponent](./components/pistolero.md)
3. [Componente: TablaDespachoComponent](./components/tabla-despacho.md)
4. [Componente: FiltroFechasComponent](./components/filtro-fechas.md)
5. [Componente: CargaMasivaComponent](./components/carga-masiva.md)
6. [Servicio: DespachoPdfService](./service/despacho-pdf-service.md)

---

## 1. Vista: RelacionDespachoComponent

**Selector:** `app-relacion-despacho`  
**Ubicación:** `src/app/views/logistica/relacion-despacho/relacion-despacho.component.ts`  
**Acceso:** Logística → Relación de Despacho

---

### ¿Qué hace? (para el usuario)

Esta pantalla sirve para organizar y registrar las guías que van a ser entregadas al transportador. Cuando la abres, aparece una lista con las guías que todavía faltan por despachar y te ayuda a trabajar con ellas de forma más rápida.

Puedes:

- **Registrar guías una por una** con el lector de barras o escribiéndolas manualmente.
- **Cargar muchas guías al mismo tiempo** desde un archivo de Excel.
- **Buscar y filtrar** las guías por tienda o transportadora.
- **Elegir cuáles guías** quieres incluir al generar el documento final.
- **Crear un PDF** con la relación de despacho, agrupado por transportadora.
- **Ver el historial** de despachos por fechas.
- **Limpiar** los registros del día cuando sea necesario, pero solo algunos usuarios pueden hacerlo.

---

### Ruta

```
logistica/relacion-despacho
```

---

### Decoradores y configuración técnica

```typescript
@Component({
  selector: 'app-relacion-despacho',
  standalone: true,
  imports: [
    CommonModule, FormsModule,
    ColComponent, CardComponent, CardBodyComponent,
    TourGuiadoComponent, PistoleroComponent,
    TablaDespachoComponent, FiltroFechasComponent, CargaMasivaComponent,
  ],
  templateUrl: './relacion-despacho.component.html',
  styleUrl: './relacion-despacho.component.scss',
})
export class RelacionDespachoComponent implements OnInit
```

---

### Interfaz central del módulo

Toda la información de despacho en memoria sigue este contrato, derivado de la colección `InventarioAdmision`:

```typescript
export interface RegistroDespacho {
  _id?: string;           // ID de MongoDB (presente al consultar, ausente al insertar)
  Guia: number;           // Número de guía de 12 dígitos (Number, igual que PedidosInter.Numeropreenvio)
  NombreTienda: string;   // Tienda que generó el pedido
  TotalRecaudo: string;   // Valor a recaudar al entregar
  Cliente: string;        // Nombre completo del destinatario
  Departamento: string;   // Departamento de destino
  Fecha: string;          // Timestamp ISO 8601 en UTC-5 (Colombia)
  UsuarioEmail: string;   // Email del usuario — permite filtrar solo sus registros
  DespachoGenerado: boolean; // false = pendiente (Hoy), true = despachado (Histórico)
  Productos?: string[];   // Nombres de productos del pedido
  pedidoId?: string;      // _id en PedidosInter (solo carga masiva, para fire-and-forget)
  Transportadora?: string;// Enriquecido en memoria desde PedidosInter. No se persiste.
}
```

---

### Propiedades clave

| Propiedad | Tipo | Descripción |
|---|---|---|
| `rows` | `RegistroDespacho[]` | Guías del día (`DespachoGenerado !== true`) |
| `rowsHistorico` | `RegistroDespacho[]` | Guías del rango consultado (`DespachoGenerado = true`) |
| `seleccionadas` | `RegistroDespacho[]` | Guías marcadas con checkbox; vacío = exportar todas las visibles |
| `transportadoraFiltro` | `string \| null` | Filtro activo de transportadora (`null` = todas) |
| `tiendaFiltro` | `string \| null` | Filtro activo de tienda (`null` = todas) |
| `transportadorasDisponibles` | `string[]` | Transportadoras únicas de la pestaña activa (con cascada) |
| `tiendasDisponibles` | `string[]` | Tiendas únicas de la pestaña activa (con cascada) |
| `procesando` | `boolean` | Bloquea el pistolero mientras se procesa una guía |
| `showCargaMasiva` | `boolean` | Abre/cierra el modal de carga masiva |
| `tourAbierto` | `boolean` | Abre/cierra el tour guiado |
| `puedeLimpiar` | `boolean` | `true` si el rol del usuario está en `ROLES_ADMIN` |

**Getters computados:**

| Getter | Retorna | Lógica |
|---|---|---|
| `rowsFiltrados` | `RegistroDespacho[]` | Aplica `tiendaFiltro` y `transportadoraFiltro` sobre la fuente activa |
| `activeTab` (getter) | `'Hoy' \| 'historico'` | Devuelve la pestaña actual |
| `activeTab` (setter) | — | Resetea filtros, busquedas y dropdowns; carga histórico si es la primera vez |

---

### Servicios y endpoints

| Servicio | Uso |
|---|---|
| `ConsumoGenericoService` | Consultas, inserciones, actualizaciones y eliminaciones |
| `DecompressionService` | Descomprime respuestas Zstd del backend |
| `DespachoPdfService` | Genera el PDF agrupado por transportadora |

**Endpoints por colección:**

| Colección | Operación | Cuándo |
|---|---|---|
| `InventarioAdmision` | GET (filtrado por `UsuarioEmail`) | Al cargar Hoy y Histórico |
| `InventarioAdmision` | POST | Al pistolerar o completar carga masiva |
| `InventarioAdmision` | PUT (`DespachoGenerado: true`) | Al exportar PDF desde pestaña Hoy |
| `InventarioAdmision` | DELETE (por `_id`) | Al limpiar registros del día |
| `PedidosInter` | GET (por `Numeropreenvio`) | Al pistolerar (buscar pedido) y enriquecer transportadora |
| `PedidosInter` | PUT (`ValidacionInventario`) | Fire-and-forget tras pistolerar o carga masiva |
| `Productos` | GET (por `idproducto`) | Al pistolerar, para obtener nombres de productos |

---

### Flujo principal

```
ngOnInit()
  └─► fetchTableData()
        ├─► GET InventarioAdmision (filtrado por UsuarioEmail)
        ├─► filtra en memoria: Fecha = hoy AND DespachoGenerado !== true
        └─► enriquecerTransportadoras() [Promise.all paralelo]
              ├─► GET PedidosInter por cada Guia → r.Transportadora
              └─► recalcularFiltros() → transportadorasDisponibles + tiendasDisponibles

Usuario pistolera una guía
  └─► procesarGuia()
        ├─► buscarPedido()       → GET PedidosInter
        ├─► validarGuiaEnMemoria() → bloqueante o advertencia
        ├─► guiaYaExiste()       → GET InventarioAdmision (duplicado)
        ├─► obtenerNombresProductos() → GET Productos por cada IdStock1..12
        ├─► POST InventarioAdmision
        ├─► PUT PedidosInter (fire-and-forget)
        └─► fetchTableData() [refresca tabla]

Usuario selecciona filtros (tienda / transportadora)
  └─► seleccionarTienda() / seleccionarTransportadora()
        ├─► actualiza filtro activo
        └─► recalcularFiltros() → actualiza listas en cascada

Usuario exporta PDF
  └─► exportToPDF()
        ├─► fuente = seleccionadas.length ? seleccionadas : rowsFiltrados
        ├─► DespachoPdfService.generarPDF(fuente)
        └─► [solo pestaña Hoy] PUT DespachoGenerado: true por cada registro
              └─► fetchTableData() [los registros desaparecen de Hoy]
```

---

### Filtros en cascada

Los dropdowns de **Tienda** y **Transportadora** son interdependientes:
- Al seleccionar una tienda, `transportadorasDisponibles` recalcula usando solo los registros de esa tienda.
- Al seleccionar una transportadora, `tiendasDisponibles` recalcula usando solo los registros de esa transportadora.
- Cada dropdown tiene buscador interno con `[(ngModel)]`.
- El botón muestra `×` cuando hay filtro activo para limpiarlo sin abrir el menú.

---

### Roles y permisos

```typescript
export const ROLES_ADMIN: string[] = ['Desarrollador', 'CEO', 'COO'];
```

Solo estos roles ven el botón **Limpiar** y pueden eliminar registros del día con `DespachoGenerado = false`.

---

### Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-07-03 | Adalberto González | Creación del módulo: pistolero, tabla, PDF agrupado por transportadora |

---

### Observaciones

- `Transportadora` se enriquece en memoria después de cargar la tabla y **no se persiste** en `InventarioAdmision`. Si la tabla se recarga, el enriquecimiento vuelve a ocurrir.
- El PDF se genera con `jsPDF` + `html2canvas` dentro de un `<iframe>` aislado para evitar interferencias de estilos globales.
