## Autor: Yadilton Torres

Fecha creacion: 2026-11-24

Estado: preprod

Tipo: vista

# Costos Transporte Tienda

**Selector:** `app-costostransportetienda`
**Ubicación:** `src/app/views/director-de-operaciones/costostransportetienda`
**Ruta:** `/app/director-de-operaciones/costostransportetienda` (también desde CEO)
**Rama:** `feature/costos-transporte-tienda`

---

## ¿Qué hace?

Es la pantalla del sistema **legacy** de tarifas: cuánto se le cobra a cada tienda por transportadora, en entrega y en devolución (6 fletes en pesos y 3 porcentajes). Las transportadoras que liquidan con el motor Legacy (Interrapidísimo, Envía, Servientrega, D2E) se configuran aquí; las del motor Dinámico van en [Costos Transportadora](../costos-transportadora/costos-transportadora.md).

Tiene dos pestañas, sin cambio de URL:

| Pestaña | Qué muestra |
|---|---|
| **Costos registrados** | Tabla de la colección `CostosTransporteTienda`, con buscador, selector de columnas, paginación y botón **Actualizar ›** por fila |
| **Cobertura por tienda** | Solo consulta: qué tiendas tienen sus 8 costos (entrega y devolución × 4 transportadoras) y cuáles les faltan. Ver [Cobertura](components/cobertura-costos.md) |

El botón ↻ de la cabecera recarga la pestaña que se está viendo. **+ Crear costo transporte tienda** abre el modal de crear (desde Cobertura, primero vuelve a Costos registrados).

---

## Cómo se guarda un registro

Colección `CostosTransporteTienda`:

```json
{
  "Tienda": "GAME CENTER WORLD",
  "IdTienda": "157174",
  "FleteUrbano": 9520,
  "FleteRegional": 11200,
  "FleteNacional": 14700,
  "FleteReexpedido": 36750,
  "FleteNacionalMetropolitano": 14700,
  "FleteNacionalMunicipal": 14700,
  "PorcentajeKiloAdicional": 35,
  "PorcentajeRecaudo": 4,
  "PorcentajeSeguro": 2.5,
  "Estado": "ACTIVO",
  "Transportadora": "Envia",
  "IdTransportadora": "10002",
  "EstadoGuia": "DEVOLUCION",
  "FechaCreacion": "2026-11-20"
}
```

- Los **fletes** son pesos enteros, sin puntos ni comas.
- Los **porcentajes** pueden llevar decimal con punto (`2.5`).
- `IdTienda` e `IdTransportadora` son texto. Se comparan siempre como texto.
- La llave de un costo es **tienda + transportadora + tipo de guía** (no puede repetirse).

---

## Estructura de archivos

```
costostransportetienda/
├── costostransportetienda.component.ts / .html / .spec.ts
├── components/cobertura-costos/            ← pestaña Cobertura (solo consulta)
│   ├── cobertura-costos.component.ts / .html / .spec.ts
│   └── styles/cobertura-costos.scss + .mobile.scss
├── helpers/
│   ├── cobertura.helper.ts + .spec.ts      ← cálculo de cobertura
│   ├── crear-costo.rules.ts + .spec.ts     ← validaciones del modal de crear
│   └── formato.helper.ts + .spec.ts        ← dinero, %, tipo de guía, búsqueda
├── models/costos-transporte.models.ts
└── styles/costostransportetienda.scss + .mobile.scss
```

| Pieza | Documento |
|---|---|
| Cobertura por tienda | [cobertura-costos.md](components/cobertura-costos.md) |
| Validaciones del modal de crear | [crear-costo-rules.md](helpers/crear-costo-rules.md) |
| Cálculo de cobertura | [cobertura-helper.md](helpers/cobertura-helper.md) |
| Formatos | [formato-helper.md](helpers/formato-helper.md) |
| Modelos | [costos-transporte-models.md](modelos/costos-transporte-models.md) |

---

## Servicios y endpoints

| Servicio | Llamada | Cuándo |
|---|---|---|
| `ConsumoGenericoService.consultarGenerico` | `metodoGenerico?coleccion=Tienda&Estado=ACTIVO&Integracion=Interrapidisimo` | Al iniciar: tiendas para la lista del modal de crear |
| `ConsumoGenericoService.consultarGenerico` | `metodoGenerico?coleccion=Transportadora` | Al iniciar |
| `ConsumoGenericoService.consultarGenerico` | `metodoGenerico?coleccion=CostosTransporteTienda` | Al iniciar, al crear y con ↻ |
| `ConsumoGenericoService.insertarGenerico` | `metodoGenerico?coleccion=CostosTransporteTienda` | Al confirmar **Crear costo** (`createNewCost`) |
| `ActualizacionCamposComponent` → `updateCampos` | `metodoGenerico?coleccion=CostosTransporteTienda` | Al confirmar **Guardar cambios** en el modal de actualizar |

Las respuestas vienen comprimidas y se abren con `DecompressionService.decompressGzip`.

> El rediseño **no cambió** la lógica de envío: `onCreateOrModify`, `checkForDuplicateTienda`, `createNewCost`, `fetchTableData` y `updateCampos` siguen iguales. Lo nuevo solo revisa antes de llamar a esas funciones.

---

## Modal de crear

- Listas propias de **tienda** (con buscador por nombre o Id, ignora espacios de más) y **transportadora**. La lista de tiendas solo trae las que tienen integración con Interrapidísimo.
- Segmentados **Entrega / Devolución** y **Activo / Inactivo**.
- 6 fletes (`$`) y 3 porcentajes (`%`). Los campos arrancan vacíos; el ejemplo va en el título de cada sección.
- Errores debajo de cada campo y aviso rojo si el costo ya existe.
- Alertas `lg-swal`: «Revisa antes de crear» (lista de problemas), «¿Crear este costo?» (resumen) y «¿Salir sin crear?».
- Las reglas están en [crear-costo.rules](helpers/crear-costo-rules.md).

## Modal de actualizar

Usa el componente compartido `ActualizacionCamposComponent`. Su **modo mejorado** solo se activa cuando `nombremetodo` trae `coleccion=CostosTransporteTienda`; el resto de tablas que usan ese modal siguen igual.

- Cabecera con nombre de la tienda y pastillas (Id, transportadora, tipo, estado).
- Campos por secciones: Tienda y transportadora / Fletes / Porcentajes / Estado / Otros.
- El campo cambiado se marca «MODIFICADO» y el pie cuenta los cambios.
- «¿Guardar los cambios?» muestra *campo antes → ahora*; «¿Salir sin guardar?» deja el foco en «Seguir editando».
- Los grupos se arman una sola vez en `ngOnInit` (si se recalculan en cada tecla, Angular redibuja los campos y no deja escribir).

---

## Diseño

- Cabecera blanca con ícono, título, insignia «4 transportadoras» y descripción.
- Tabla propia (ya no usa `app-tables`): tienda con Id debajo, fletes en `$` con formato `es-CO`, porcentajes con `%`, estado en pastilla (Activo verde, Inactivo gris).
- Paginación con `app-pie-paginacion modo="lista"`: «Mostrar 10 ▾», números con saltos (`1 2 3 … 352`). 10 registros en computador y 5 en celular. Sin scroll interno.
- **Tema «La Red»:** la vista lleva `lg-vista-noche` y el modal `lg-noche`. Los colores usan tokens `--lg-*` con el color anterior como respaldo (`var(--lg-texto, #0f172a)`), así se ve bien en modo oscuro y en modo claro.
- **Celular** (`styles/*.mobile.scss`, cortes 860 / 768 / 380 px): la tabla pasa a tarjetas, filtros en hoja a pantalla completa, modales a pantalla completa con controles de 44 px.
- Pesos de letra: solo 500, 600 y 700.

---

## Pruebas

| Archivo | Qué cubre |
|---|---|
| `costostransportetienda.component.spec.ts` | Con servicios falsos: consulta las mismas colecciones de siempre, formatos, validaciones, pegado, teclas, duplicado y ↻ |
| `helpers/crear-costo.rules.spec.ts` | Todas las reglas del modal de crear |
| `helpers/formato.helper.spec.ts` | Dinero, tipo de guía, títulos, búsqueda |
| `helpers/cobertura.helper.spec.ts` | Cálculo de cobertura (7 casos) |
| `components/cobertura-costos/cobertura-costos.component.spec.ts` | Solo consulta y aviso si falla |

Junto con las pruebas de `pie-paginacion` y `actualizacion-campos`: **57 pruebas, todas en verde** (2026-11-24).

```bash
npx ng test --watch=false --browsers=ChromeHeadlessCI \
  --include='src/app/views/director-de-operaciones/costostransportetienda/**/*.spec.ts'
```

---

## Pendientes

- Al crear, el mensaje original dice «Pago creado exitosamente» (es lógica vieja; no se cambió).
- `fetchTableData()` se llama dos veces al crear (lógica vieja).
- Datos: 5 tarifas genéricas (`IdTienda 00000`) incompletas; ver [Cobertura](components/cobertura-costos.md#tarifas-genericas).
- Faltan los costos de D2E por tienda.
