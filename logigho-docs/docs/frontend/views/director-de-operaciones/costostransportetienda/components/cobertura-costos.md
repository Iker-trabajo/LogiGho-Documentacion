## Autor: Yadilton Torres

Fecha creacion: 2026-11-24

Estado: preprod

## Componente: Cobertura por tienda

**Selector:** `app-cobertura-costos`
**Ubicación:** `components/cobertura-costos/`
**Tests:** `cobertura-costos.component.spec.ts`
**Lógica:** [cobertura.helper](../helpers/cobertura-helper.md)

---

## ¿Qué hace?

Pestaña de **solo consulta**: responde «¿qué tiendas tienen todos sus costos y a cuáles les falta algo?». No crea ni edita nada.

Cada tienda necesita **8 costos activos**: entrega y devolución en Interrapidísimo (10001), Envía (10002), Servientrega (10004) y D2E (10005). TCC no entra porque liquida con el motor Dinámico.

- Un costo **inactivo** cuenta como faltante.
- Las **tarifas genéricas** (`IdTienda 00000`) no cuentan para ninguna tienda.
- Los Id se comparan como texto.

---

## Qué consulta

Al abrir y con ↻ (en paralelo):

| Colección | Para qué |
|---|---|
| `Tienda` | Lista de tiendas (activas e inactivas) |
| `CostosTransporteTienda` | Los costos |
| `Transportadora` | Ids y logos de las 4 transportadoras |

Si alguna consulta falla, muestra «No se pudo cargar la cobertura de costos. Intenta de nuevo.».

---

## Partes de la pantalla

| Parte | Comportamiento |
|---|---|
| **Tarjetas de transportadora** | Logo, estado, barra (completas / solo una / sin costos) y números. Un clic filtra la tabla por esa transportadora |
| **Barra de filtros** | Buscador (lupa en trazo, como los demás módulos), segmentado Activas / Inactivas / Todas y 3 listas propias: Cobertura, Transportadora y Transp. (estado). Una sola lista abierta a la vez; se cierran con clic afuera, Esc o al elegir. «✕ Limpiar» siempre ocupa su lugar (solo se oculta) para que nada se mueva. Todo en una sola fila en computador |
| **Leyenda** | E = entrega, D = devolución; íconos ✓ activo, – inactivo, ✕ sin costo, y «Registrado más de una vez» |
| **Tabla** | Cabecera azul de una fila. Por transportadora: barrita de 2 partes (E \| D) y debajo los íconos. Columna «Resultado» (ej. «3/8 Le faltan 5») y botón **Ver ›** |
| **Aviso de cabecera** | Al pasar el mouse por el nombre de una transportadora: qué se necesita, cómo leer las casillas y la situación de hoy con números. Está oculto con `display: none` (si solo fuera invisible, estiraba la tabla y aparecía scroll con pocas filas) |
| **Detalle** | Panel lateral derecho que se agranda arrastrando; se cierra con ✕, Esc o clic afuera. Muestra los valores de cada costo y los duplicados |
| **Tarifas genéricas** | Sección plegable (ver abajo) |
| **Paginación** | `app-pie-paginacion modo="lista"`, sin scroll interno: la tabla mide lo que miden sus filas |

---

## Tarifas genéricas

Lista los registros con `IdTienda 00000` tal como están y marca lo que les falta: Completo, Sin IdTransportadora, Sin EstadoGuia o Transportadora no reconocida. Muestra los contadores «N registros» y «N por revisar».

Al 2026-11-20 había 5 por revisar (dato, no código):

- Servientrega: falta `IdTransportadora 10004` y `EstadoGuia`.
- «Generico»: no tiene transportadora.
- «D2»: probablemente es D2E (10005).

---

## Celular

`styles/cobertura-costos.mobile.scss`, cargado de último:

- Tarjetas de transportadora en fila deslizable.
- Buscador a todo el ancho y botón **☰ Filtros [n]** a todo el ancho (el ↻ está en la cabecera de la vista).
- Filtros en hoja a pantalla completa, por encima de la cabecera del sitio: secciones en tarjetas blancas, fichas, «Limpiar» y «Ver N tiendas». En la hoja, «Transp.» se lee «Estado de la transportadora».
- Leyenda en tarjeta: explicación arriba e íconos en 2×2. «Pasa el mouse…» se oculta.
- Cada tienda es una tarjeta con las 4 transportadoras en 2×2.
- Las tarjetas de transportadora no cambian de alto al filtrar: sin resultados queda una barra gris.

---

## Pruebas

`cobertura-costos.component.spec.ts` (servicios falsos):

- Se crea.
- Solo consulta `Tienda`, `CostosTransporteTienda` y `Transportadora`, en ese orden.
- Si la consulta falla, muestra el aviso y deja de cargar.
