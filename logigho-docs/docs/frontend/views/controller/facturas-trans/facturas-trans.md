---
autor: Iker
fecha_creacion: 2026-03-23
ultima_actualizacion: 2026-10-02
estado: desarrollo
nivel: 2
---

# Facturas Transportadoras

**Autor:** Iker Acevedo

**Selector:** `app-facturas-trans`

**Ubicación:** `SitioLogiGho/src/app/views/controller/facturas-trans`

---

## ¿Qué hace?

Centraliza las facturas de facturas de todas las transportadoras integradas al sistema (Interrapidísimo, Envia, X-Cargo, Servientrega). Permite consultar los registros almacenados en MongoDB, buscar por cualquier campo, importar nuevas facturas desde Excel y exportar los datos actuales a Excel.

---

## Roles y acceso

| Acceso | Descripción |
|---|---|
| Autenticado | Necesita login |
| Rol específico | Requiere rol: `controller` |

---

## Estructura de archivos

```
facturas-trans/
├── facturas-trans.component.ts
├── facturas-trans.component.html
├── facturas-trans.component.scss
└── strategies/
    └── transportadoras/
        ├── transportadora.strategy.ts     ← Contrato/interfaz base
        ├── transportadora.factory.ts      ← Registro central de transportadoras
        └── servientrega.strategy.ts       ← Implementación Servientrega
```

---

## Secciones de la página

1. **Facturación Interrapidísimo** — Tabla + búsqueda + importar + exportar (implementación legacy)
2. **Facturación Envia** — Tabla + búsqueda + importar + exportar (implementación legacy)
3. **Facturación X-Cargo** — *Fuera de servicio por el momento* (ver [Reactivar X-Cargo](#reactivar-x-cargo))
4. **Facturación Servientrega** — Tabla + búsqueda + importar + exportar (implementación con Strategy)

---

## Estado de implementación por transportadora

| Transportadora | Colección MongoDB | Patrón | Componente de importación |
|---|---|---|---|
| Interrapidísimo | `FacturacionInter` | Legacy | `app-importacion-facturas` |
| Envia | `FacturacionEnvia` | Legacy | `app-importacion-facturas-envia` |
| X-Cargo | `FacturacionXcargo` | Legacy — **fuera de servicio** | `app-importacion-facturas-xcargo` (comentado) |
| Servientrega | `FacturacionServientrega` | **Strategy** | `app-importacion-facturas-generico` |

> Las transportadoras legacy funcionan correctamente. La migración al patrón Strategy queda pendiente de evaluación con el equipo, hacer gradualmente e ir probando en produccion para no afectar por completo al sistema.

---

## Propiedades del componente

| Propiedad | Tipo | Default | Descripción |
|---|---|---|---|
| `rows` | `TablaRow[]` | `[]` | Filas visibles de Inter |
| `rowsMemory` | `TablaRow[]` | `[]` | Copia original de filas Inter para restaurar filtros |
| `rowsEnvia` | `TablaRow[]` | `[]` | Filas visibles de Envia |
| `rowsXcargo` | `TablaRow[]` | `[]` | Filas visibles de X-Cargo |
| `tablaEstadoMap` | `Map<string, TablaEstado>` | `new Map()` | Estado de tablas para transportadoras con Strategy |
| `servientregaStrategy` | `TransportadoraStrategy` | `Factory.crear('servientrega')` | Instancia única de la strategy de Servientrega |
| `isServientregaModalOpen` | `boolean` | `false` | Controla visibilidad del modal de importación de Servientrega |

---

## Métodos

### `fetchTableData(): void`
**Descripción:** Carga los datos de Interrapidísimo desde MongoDB, los descomprime y los asigna a `rows`.

**Proceso:**
1. Llama a `consultarGenerico` con colección `FacturacionInter`
2. Descomprime con `DecompressionService`
3. Sanitiza campos y genera columnas dinámicas

---

### `fetchTableDataEnvia(): void`
**Descripción:** Mismo proceso que `fetchTableData` pero para la colección `FacturacionEnvia`.

---

### `fetchTableDataXcargo(): void`
**Descripción:** Mismo proceso que `fetchTableData` pero para la colección `FacturacionXcargo`.

> **Fuera de servicio:** su llamada en `ngOnInit()` está comentada desde 2026-10-02. El método sigue existiendo.

---

### `fetchTableDataByStrategy(strategy: TransportadoraStrategy): void`
**Descripción:** Método genérico que carga datos de cualquier transportadora registrada en el Factory usando su strategy.

**Parámetros:**
- `strategy` — Objeto con la configuración y lógica de la transportadora (colección, campos, sanitize)

**Proceso:**
1. Llama a `consultarGenerico` usando `strategy.coleccion`
2. Descomprime y aplana los datos con `.flat()`
3. Sanitiza cada item con `strategy.sanitize(item)`
4. Almacena filas, columnas y estado en `tablaEstadoMap`

---

### `getRowsByStrategy(strategy: TransportadoraStrategy): TablaRow[]`
**Descripción:** Retorna las filas visibles actuales de una transportadora desde el `tablaEstadoMap`.

**Parámetros:**
- `strategy` — Objeto con la configuración y lógica de la transportadora

---

### `getColumnsByStrategy(strategy: TransportadoraStrategy): TablaColumn[]`
**Descripción:** Retorna las columnas generadas de una transportadora desde el `tablaEstadoMap`.

**Parámetros:**
- `strategy` — Objeto con la configuración y lógica de la transportadora

---

### `getTotalMontoByStrategy(strategy: TransportadoraStrategy): number`
**Descripción:** Suma el campo total de todas las filas de una transportadora. El campo que suma está definido en `strategy.campoTotal`.

**Parámetros:**
- `strategy` — Objeto con la configuración y lógica de la transportadora

---

### `filterByStrategy(strategy: TransportadoraStrategy, valorBuscado: string): void`
**Descripción:** Filtra las filas de una transportadora por texto. Si el valor está vacío, restaura desde `rowsMemory`.

**Parámetros:**
- `strategy` — Objeto con la configuración y lógica de la transportadora
- `valorBuscado` — Texto ingresado por el usuario en el input de búsqueda

---

### `exportByStrategy(strategy: TransportadoraStrategy): void`
**Descripción:** Exporta las filas actuales de una transportadora a Excel usando `strategy.exportFileName` como nombre del archivo.

**Parámetros:**
- `strategy` — Objeto con la configuración y lógica de la transportadora

---

## Flujo principal

```
ngOnInit()
  ├── fetchTableData()           → carga Inter (legacy)
  ├── fetchTableDataEnvia()      → carga Envia (legacy)
  ├── // fetchTableDataXcargo() → X-Cargo fuera de servicio (comentado)
  └── TransportadoraFactory.obtenerTodas().forEach(strategy =>
        fetchTableDataByStrategy(strategy))  → carga Servientrega y futuras
```

---

## Cómo agregar una nueva transportadora

> Seguir este proceso para cada transportadora nueva. No modificar código existente.

**Paso 1 — Crear la strategy**

Crear `nueva-transportadora.strategy.ts` en `strategies/transportadoras/`:

```typescript
export class NuevaTransportadoraStrategy implements TransportadoraStrategy {
  coleccion         = 'FacturacionNueva';
  nombre            = 'NUEVA TRANSPORTADORA';
  exportFileName    = 'Facturacion_Nueva';
  campoTotal        = 'VALOR_TOTAL';
  plantillaFileName = 'Plantilla_facturas_nueva.xlsx';
  campoGuia         = 'NUMEROGUIA';

  columnMapping = {
    'CAMPO_EXCEL': 'CAMPO_DB',
    // ...
  };

  // Opcional: columnas del columnMapping que NO son obligatorias en el Excel.
  // Si no se define, todas las columnas del columnMapping son obligatorias.
  columnasOpcionales: string[] = ['CAMPO_EXCEL_OPCIONAL'];

  sanitize(item: any) {
    return {
      CAMPO: item.CAMPO ?? 'N/A',
      VALOR: item.VALOR ?? 0,
    };
  }
}
```

**Paso 2 — Registrar en el Factory**

En `transportadora.factory.ts`, agregar una línea:

```typescript
private static readonly estrategias = {
  servientrega: new ServientregaStrategy(),
  nueva:        new NuevaTransportadoraStrategy(), // ← agregar aquí
};
```

**Paso 3 — Agregar estado en el componente `.ts`**

```typescript
readonly nuevaStrategy = TransportadoraFactory.crear('nueva');
isNuevaModalOpen = false;
openNuevaModal()  { this.isNuevaModalOpen = true;  }
closeNuevaModal() { this.isNuevaModalOpen = false; }
```

**Paso 4 — Agregar sección en el HTML**

Copiar el bloque de Servientrega en el HTML y reemplazar `servientregaStrategy` por `nuevaStrategy`.

> La validación de encabezados del componente genérico usa el `columnMapping` de la strategy y respeta `columnasOpcionales`. Ver [Importación Facturas Genérico](../../../components/importacion-facturas-generico.md#validacion-de-encabezados).

---

## Reactivar X-Cargo

X-Cargo está **fuera de servicio por el momento** (ya no se usa en la empresa). Se comentó con la nota `FUERA DE SERVICIO POR EL MOMENTO` en dos archivos de este módulo. La carpeta `src/app/components/importacion-facturas-xcargo/` no se tocó.

| Archivo | Qué se comentó |
|---|---|
| `facturas-trans.component.html` | El bloque "FACTURACIÓN X-CARGO" (líneas 56 a 82 aprox.) |
| `facturas-trans.component.ts` | El `import` de `ImportacionFacturasXcargoComponent` |
| `facturas-trans.component.ts` | Su entrada en la lista `imports` del `@Component` |
| `facturas-trans.component.ts` | La llamada `this.fetchTableDataXcargo()` en `ngOnInit()` (evita una consulta inútil a la colección `FacturacionXcargo`) |

```typescript
// FUERA DE SERVICIO POR EL MOMENTO (X-Cargo ya no se usa en la empresa). Descomentar junto con el HTML y ngOnInit si se necesita de nuevo.
// import { ImportacionFacturasXcargoComponent } from '../../../components/importacion-facturas-xcargo/importacion-facturas-xcargo.component';
```

> **Para reactivarlo hay que descomentar las cuatro cosas a la vez**, o falla la compilación.

---

## Dependencias externas

| Librería | Versión | Uso |
|---|---|---|
| XLSX (SheetJS) | ^0.18 | Lectura y escritura de archivos Excel |
| file-saver | ^2.x | Descarga de archivos en el navegador |
| sweetalert2 | ^11.x | Alertas y confirmaciones |

## Servicios Angular utilizados

| Servicio | Métodos usados | Propósito |
|---|---|---|
| `ConsumoGenericoService` | `consultarGenerico()`, `insertarGenerico()` | Lectura y escritura en MongoDB |
| `DecompressionService` | `decompressGzip()` | Descompresión de datos paginados del API |

---

## Estilos

### Variables SCSS principales

| Variable | Valor | Uso |
|---|---|---|
| Gradiente principal | `#121B60 → #3d4fd6` | Botones y encabezado del módulo |
| Fondo totales | `#f8f9fa` | Barra inferior de totales por sección |

### Animaciones

| Animación | Duración | Uso |
|---|---|---|
| `fadeInLeft` | 0.5s | Entrada de elementos al cargar |

---

## Subcomponentes

| Componente | Selector | Descripción |
|---|---|---|
| [`ImportacionFacturasComponent`](../../../components/importacion-facturas.md) | `app-importacion-facturas` | Modal importación Inter (legacy) |
| [`ImportacionFacturasEnviaComponent`](../../../components/importacion-facturas-envia.md) | `app-importacion-facturas-envia` | Modal importación Envia (legacy) |
| `ImportacionFacturasXcargoComponent` | `app-importacion-facturas-xcargo` | Modal importación X-Cargo (legacy) — fuera de servicio, comentado |
| [`ImportacionFacturasGenericoComponent`](../../../components/importacion-facturas-generico.md) | `app-importacion-facturas-generico` | Modal importación genérico — recibe `TransportadoraStrategy` como `@Input()` |
| `TablesComponent` | `app-tables` | Tabla reutilizable con columnas dinámicas |

---

## Changelog del componente

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-03-23 | Iker | Implementación de patrón Strategy para Servientrega. Creación de `TransportadoraStrategy`, `TransportadoraFactory` y `ServientregaStrategy`. Creación de `app-importacion-facturas-generico`. |
| 2026-05-02 | Adalberto | Integración de `D2EStrategy` en el módulo de facturación. Creación de `d2e.strategy.ts` con `columnMapping`, `sanitize()` y `transformarValor()` para normalización de fechas seriales de Excel a formato `dd-mm-yyyy`. Registro en `TransportadoraFactory`. |
| 2026-05-02 | Adalberto | Agregado `transformarValor?(campo, valor)` como método opcional en la interfaz `TransportadoraStrategy`. Permite a cada strategy manejar transformaciones de tipo por campo al importar Excel sin modificar el componente genérico. |
| 2026-05-02 | Adalberto | Corrección en `exportToExcel`: se excluye el campo `_id` del dataset antes de generar el worksheet para que no aparezca en los archivos Excel exportados. |
| 2026-05-02 | Adalberto | Corrección de búsqueda con clic repetido en D2E: el input usaba `[ngModel]` unidireccional, por lo que `searchValueD2E` nunca se actualizaba y el botón Buscar pasaba siempre `''`, reseteando el filtro. Corregido a `[(ngModel)]`. |
| 2026-10-02 | Yadilton Torres | Validación de encabezados en la importación de facturas (Envía, Inter y genérico). Agregado el campo opcional `columnasOpcionales?: string[]` a `TransportadoraStrategy`; `ServientregaStrategy` lo usa para `CODIGOCENTROCOSTO` y `NOMBRECENTROCOSTO`. |
| 2026-10-02 | Yadilton Torres | X-Cargo fuera de servicio: se comentó el bloque HTML "FACTURACIÓN X-CARGO", el import de `ImportacionFacturasXcargoComponent`, su entrada en `imports` y la llamada a `fetchTableDataXcargo()` en `ngOnInit`. |

---

## Observaciones

- Las transportadoras Interrapidisimo, Envia y X-Cargo mantienen su implementación legacy. Migración al patrón Strategy pendiente de evaluación con el equipo.
- X-Cargo está comentado (fuera de servicio). Ver [Reactivar X-Cargo](#reactivar-x-cargo).
- Los tres componentes de importación rechazan los archivos sin fila de encabezados o con columnas desconocidas. Un archivo con columnas fuera del `columnMapping` ya no se guarda con el nombre que traiga el Excel.
- El `tablaEstadoMap` centraliza el estado de todas las transportadoras que usen Strategy. Las legacy tienen variables individuales (`rows`, `rowsEnvia`, `rowsXcargo`).
- La colección `FacturacionServientrega` define su schema con la primera inserción. Los campos están definidos en `ServientregaStrategy.columnMapping`.
