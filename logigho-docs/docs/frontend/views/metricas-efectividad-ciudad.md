# Entregas — "Efectividad por Ciudad" (vista de usuarios)

Vista de **Mi Tienda** que le muestra a cualquier usuario, para cada municipio, **qué transportadora entrega mejor** (historia 016). Lee el resumen que dejó la última corrida de la [lambda de métricas](../../backend/metricas/index.md). El panel de quien administra la métrica está en [Configuración de Entregas](metricas-configuracion-entregas.md).

| | |
|---|---|
| **Ruta** | `/app/tienda/entregas` |
| **Módulo de menú** | `Modulo` ID `169` — `Entregas \| Mi Tienda` (padre `1`), icono `cilTruck`; mismos roles que Pedidos (ID 19) |
| **Código** | `SitioLogiGho/src/app/views/tienda/entregas/` |
| **Backend** | `GET {apiUrlRest}metricas/entrega-ciudad` |
| **Tests** | `ng test`: 118 en verde entre esta vista y el panel de administración |

## Qué ve el usuario

0. **Ciudad por defecto:** la vista abre **siempre en Medellín** (se busca por su código DANE `05001000` y, si no está, por nombre; sin Medellín, la primera ciudad). Decisión de Iker, 2026-10-30.
1. **Encabezado** (sin miga de pan, pegado a la cabecera de la plataforma) con el título, el subtítulo y, arriba a la derecha, **"Actualizado en 29 de oct de 2026, 11:54 a. m."** (fecha de corte de la última corrida finalizada).
2. **Consulta rápida de efectividad municipal:** un buscador tipo *command palette*. Al escribir una ciudad se listan coincidencias con la mejor transportadora, su porcentaje y los despachos; se navega con ↑ ↓, `Enter` y `Esc`.
3. **Tarjeta de la ciudad elegida:** a la izquierda la **mejor opción** (la transportadora de mayor porcentaje entre las que tienen muestra suficiente), con su porcentaje, su muestra y el botón *Priorizar*; si ninguna alcanza el mínimo muestra "Sin recomendación". A la derecha, el **Ranking comparativo de transportadoras** de esa ciudad: una fila por transportadora, de mejor a peor, con posición, sigla, guías, porcentaje y barra. Una fila bajo 80 % lleva la insignia "⚠ Revisar". Las transportadoras con menos guías que el mínimo van aparte, punteadas, como **"Pocos datos (N guías)"**.
4. **Todos los municipios:** tabla nacional con las columnas de cada transportadora, la mejor opción por fila y estos controles:
   - búsqueda por municipio;
   - **Departamento** y **Ordenar por** (mayor/menor efectividad, mayor volumen, A-Z) como dropdowns propios (el de departamento dice solo **"Todos"**, sin conteo) que ocupan todo el ancho de su columna (en pantallas medianas el buscador ocupa la fila y los dos dropdowns quedan lado a lado; en móvil se apilan);
   - **chips de transportadora** (multiselección): filtran columnas y recalculan mejor opción, semáforo y orden sobre lo elegido;
   - **leyenda de semáforo clicable** como filtro;
   - paginación (25, 50 o 100 por página), con el total de municipios y de guías finales en el título;
   - **Exportar Excel** (CSV con BOM, abre con tildes correctas en Excel).

El buscador de la parte superior muestra, por cada coincidencia, solo **"★ Mejor opción: <transportadora>"** en un color neutro (sin porcentaje ni rojos que preocupen).

### Qué significan los datos en gris

| En la tabla | Significa |
|---|---|
| `ⓘ 100 %` en gris | **Muestra insuficiente:** hay porcentaje, pero con menos guías con resultado final que el mínimo (30 por defecto). No es confiable: no cuenta para la mejor opción ni para los rankings |
| `—` punteado | Esa transportadora **no tiene guías con resultado final** en esa ciudad |

### Hover (tooltips)

Al pasar el cursor aparece **al instante** un tooltip propio (fondo oscuro, texto corto) que explica cada dato: las celdas de la tabla (nivel del semáforo y con cuántas guías se calculó, o por qué están en gris), los encabezados de columna, "Mejor opción", "No concluyente", las filas del ranking, los niveles de la leyenda, los chips de transportadora, "Muestra auditada", "Entrega efectiva" y "Actualizado en". Es la directiva `egTip` (`components/eg-tip`): se coloca encima del dato (o debajo si no hay espacio), no se sale de la pantalla y no lo recorta la tabla con scroll. Reemplaza al `title` del navegador, que tardaba en aparecer. Los textos **nunca mencionan el porcentaje real**, solo lo que se muestra.

### Semáforo

| Nivel | Regla |
|---|---|
| Excelente | 90 % o más |
| Aceptable | 80 a 89 % |
| Crítico | menos de 80 % |
| Sin muestra | menos guías finales que el mínimo de la configuración (30 por defecto) |

El filtro de la leyenda es **por celda**: un municipio pertenece a un nivel si alguna de sus transportadoras con muestra cae en él. Los umbrales 90/80 son los de la vista (decisión de Iker, 2026-10-29); la spec 004 había propuesto 40/60.

### Estados que no son datos

- **Cargando**, con indicador.
- **Error:** si falla la carga, un mensaje según el código (sesión vencida, sin permiso, sin conexión, genérico) y el botón **Reintentar**.
- **Sin datos:** "Aún no hay datos calculados…" si todavía no existe una corrida finalizada.

## Cómo se arman los datos

`MetricasEntregaService` (`services/`) consume el REST del entorno y **traduce** la respuesta cruda a la forma de la vista:

- Lee las claves en **PascalCase o camelCase** (la lambda no usa camelCase).
- Convierte el diccionario `porTransportadora` en una lista por ciudad y arma las **columnas dinámicas** desde `transportadoras` del backend.
- **TCC se oculta** (`OCULTAS`) hasta que sus guías estén en `PedidosInter` (Iker, 2026-10-29). Cuando lleguen, se quita de esa lista y se agregan su nombre y sigla.
- La vista **no tiene selector de período**: muestra todo lo que dejó calculado la corrida; el alcance lo define la corrida, no el usuario (M-146).
- `obtenerResumenParaUsuario()` aplica además el [ajuste visual](../../backend/metricas/ajuste-visual.md) (si está activo) antes de pintar. El resto de la vista trabaja con esos números.

## Funciones puras (con tests)

`utils/entrega.utils.ts` (semáforo, mejor transportadora, ranking comparativo `separarRanking`, filtro y orden de la tabla, sugerencias del buscador, total de guías, restricción por transportadoras, textos del hover, mensajes de error) y `utils/ajuste.utils.ts` (regla del ajuste). Sin dependencias de Angular, probadas con `entrega.utils.spec.ts` y `ajuste.utils.spec.ts`. El dropdown propio es `components/eg-select`.

## Entornos

`apiUrlRest` vive en los `environment`: desarrollo y PreProd apuntan al REST `8u5isw5u30` (stage `PreProd`); **producción** (`environment.ts`) al REST `7elgyten88` (stage `Produccion`).

!!! warning "Discrepancia corregida (2026-10-30)"
    `environment.ts` (el build de producción) tenía `apiUrlRest` apuntando al gateway de **PreProd**: Entregas y el panel habrían leído PreProd desde producción. Se corrigió antes del despliegue (commit `9c00ff5e`).

Para ver datos de producción en local: `ng serve --configuration production`. **Ojo:** toda la app queda contra producción; usar solo para mirar.

## Pendientes conocidos

- El botón **Priorizar** navega a `/app/tienda/pedidos?ciudad=&mun8=&transportadora=`, pero Pedidos todavía **no lee** esos parámetros.
- No hay una nota visible de que el porcentaje puede estar ajustado (recomendado, no implementado).
- Orden por cualquier columna de la tabla, ranking de peores ciudades y la prueba de responsive a fondo (M-138): sin hacer.
- Hay specs preexistentes de otros componentes que fallan por no proveer `HttpClient` y un aviso NG8107 de `modal-creacion-pedidos`: deuda ajena a esta historia.
