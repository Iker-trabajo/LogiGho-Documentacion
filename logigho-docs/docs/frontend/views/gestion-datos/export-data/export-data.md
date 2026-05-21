---

## Autor: Adalberto González

Fecha creacion: 2026-05-13
Estado: produccion
Tipo: vista

# Vista: ExportDataComponent

**Selector:** `app-export-data`  
**Ubicación:** `src/app/views/export-data/export-data.component`  
**Acceso:** Autenticado | Guard: `AuthGuard`

---

## ¿Qué hace?

Pantalla principal del módulo de exportación. Permite al usuario seleccionar una colección de MongoDB, aplicar filtros avanzados por campo, definir un rango de fechas, elegir las columnas a incluir y el formato de salida (Excel o PDF). Al confirmar, abre una nueva pestaña con `ExportRunnerComponent` pasando toda la configuración via URL y `localStorage`.

---

## Ruta

| Ruta | Guard | Parámetros de URL |
| --- | --- | --- |
| `/app/export-data` | `AuthGuard` | — |

---

## Propiedades clave

| Propiedad | Tipo | Descripción |
| --- | --- | --- |
| `colecciones` | `Coleccion[]` | Lista de colecciones disponibles, cargada desde `ListaColecciones` al iniciar |
| `campoConfigs` | `CampoConfig[]` | Configuración de campos de la colección activa, cargada desde `ConfiguracionCamposColeccion`. Determina tipo de control, opciones, orden y visibilidad de cada campo en los filtros |
| `filtrosAvanzados` | `FiltroAvanzado[]` | Filtros que el usuario ha agregado. Cada entrada tiene `campo` y `valor` |
| `camposMetadata` | `Map<string, CampoMetadata>` | Metadata por campo: si es fecha y su formato (`YYYY-MM-DD`, `DD/MM/YYYY`, `YYYY-MM-DD HH:mm:ss`). Se infiere de `campoConfigs` o de los datos reales de la colección |
| `campoSelectOptions` | `Map<string, {value,label}[]>` | Caché lazy de opciones para campos SELECT/MULTISELECT. Se puebla la primera vez que el usuario abre el dropdown de ese campo |
| `filtroFechaPredefinido` | `string` | Período predefinido seleccionado: `ultima_semana`, `ultimo_mes`, `ultimos_3_meses`, `ultimos_6_meses`, `todo` |
| `puedeVerFiltroTodo` | `boolean` | Si es `false` (según rol), la opción `todo` no aparece en la UI y se fuerza un rango de fechas |
| `puedeVerExportarServidor` | `boolean` | Si es `true` (según rol), se muestra el botón de exportación por servidor además del botón normal |
| `totalRegistros` | `number` | Total de registros de la colección con los filtros actuales. Se guarda en `localStorage` para que `export-runner` calcule cuántas páginas debe pedir |

---

## Servicios y endpoints

| Servicio | Método | Endpoint | Cuándo |
| --- | --- | --- | --- |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET /metodoGenerico?coleccion=ListaColecciones&Estado=ACTIVO` | `ngOnInit` — carga el listado de colecciones |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET /metodoGenerico?coleccion=ConfiguracionCamposColeccion&NombreColeccion=X&Estado=ACTIVO&mcomp=2` | Al seleccionar colección — carga la configuración de campos |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET /metodoGenerico?coleccion=X&mcomp=2&lote=1&page=1&[filtros]` | Al seleccionar colección o cambiar filtros — obtiene `TotalRegistros` |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET /metodoGenerico?coleccion=Tienda&mcomp=2&lote=500&Estado=ACTIVO` | Lazy, al abrir un SELECT/MULTISELECT cuyo `CampoEtiquetaReferencia === 'NombreTienda'` |
| `DecompressionService` | `decompressZstd()` / `decompressGzip()` | — | Para descomprimir todas las respuestas del backend |

---

## Flujo principal

```
ngOnInit()
  -> actualizarPermisosRol()
       // determina puedeVerFiltroTodo y puedeVerExportarServidor según rol en sessionStorage
  -> cargarColecciones()
       // GET ListaColecciones → puebla colecciones[]
  -> [usuario selecciona colección]
  -> onColeccionChange()
  -> cargarColumnas()
       // GET colección con lote=1, page=1 + filtros base de ListaColecciones.FiltroReferencia
       // → infiere columnas disponibles y obtiene TotalRegistros
  -> cargarConfiguracionCampos()
       // GET ConfiguracionCamposColeccion → campoConfigs[] ordenados por Orden ASC
       // → puebla camposMetadata (detecta fechas y formatos)
       // → camposDisponibles = campoConfigs.map(c => c.NombreCampo)

  -> [usuario abre dropdown de campo SELECT/MULTISELECT]
  -> ensureSelectOptions(campo)
       // si TipoOpciones === 'LISTA' → usa cfg.Opciones directamente
       // si TipoOpciones === 'COLECCION':
       //   GET coleccionReferencia con buildReferenciaQueryParamsSinValor (omite #VALOR)
       //   si CampoEtiquetaReferencia === 'NombreTienda':
       //     → usa NombreTienda como value (no Id) para que el backend reciba el nombre
       //     → filtra en cliente por tiendas_asignadas de sessionStorage

  -> [usuario agrega filtro]
  -> agregarFiltro()
       // valida que no existan duplicados de campo
       // empuja a filtrosAvanzados[]

  -> exportarDatos()
       // construye URLSearchParams con filtros base + filtrosAvanzados
       // guarda en localStorage:
       //   export_columns_<exportId>        → columnas seleccionadas
       //   export_total_registros_<exportId> → totalRegistros
       // abre window.open('/app/export-runner?coleccion=X&formato=Y&exportId=Z&[filtros]')
       // limpia entradas de localStorage con más de 5 minutos de antigüedad
```

---

## Cómo funciona el filtro de tiendas asignadas

El usuario tiene en `sessionStorage['tiendas_asignadas']` los nombres de tienda a los que tiene acceso, separados por coma (ej: `"BENDITA FRAGANCIA,BTK SHOR DAMA"`). Si contiene `"Todas"`, ve todas sin restricción.

Al cargar las opciones de un campo `COLECCION` que referencia `Tienda`, el componente:
1. Trae todas las tiendas del backend sin filtro de nombre (solo `Estado=ACTIVO`)
2. Construye los options usando siempre `NombreTienda` como `value` (no el `Id` numérico) para que el filtro enviado al backend sea el nombre
3. Filtra en cliente comparando el `label` (NombreTienda) contra la lista asignada

> **Por qué se filtra en cliente:** `ConsumoGenericoService.consultarGenerico()` usa `HttpParams.set()` internamente, que sobreescribe valores duplicados. Enviar `Tienda=A&Tienda=B` vía ese servicio no es posible. La alternativa de enviarlo en el querystring directo codifica los espacios como `%20` y el backend real no los trata como OR múltiple.

---

## Historial de cambios

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-05-13 | Adalberto González | Documentación inicial |
| 2026-05-13 | Adalberto González | Fix desfase UTC: parseo de `YYYY-MM-DD` como fecha local para evitar desfase de un día |
| 2026-05-13 | Adalberto González | Fix filtrado de tiendas asignadas: aplica a todas las colecciones por `CampoEtiquetaReferencia === 'NombreTienda'`, no solo a las que tienen `#VALOR` en `FiltroReferencia` |
| 2026-05-13 | Adalberto González | Fix `CampoValorReferencia`: se usa `NombreTienda` como value en opciones de Tienda para que el backend reciba el nombre y no el Id numérico |
| 2026-05-13 | Adalberto González | Eliminado botón "Ver Ejemplos" y su panel de filtros anidados |

---

## Observaciones

- **`ConfiguracionCamposColeccion` es la fuente de verdad** del módulo. Si un campo no aparece, aparece con el control equivocado o el selector de tiendas sale vacío, el primer lugar a revisar es esa colección en MongoDB. Los campos con `Estado: INACTIVO` son ignorados por completo.
- **Campos DATE están desactivados** (`Estado: INACTIVO` en `ConfiguracionCamposColeccion`) en todas las colecciones porque el filtro avanzado de igualdad exacta no hace match con campos que guardan datetime completo (`"2026-02-16 18:06:01"`). El filtro de rango principal (`fechasFiltro`) sí funciona correctamente porque usa el sistema propio del backend.
