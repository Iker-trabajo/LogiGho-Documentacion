# Costos transporte tienda: rediseño, cobertura y validaciones — noviembre 2026

## [2026-11-24] — Costos transporte tienda

**Estado:** probado en **PreProd** el 2026-11-23 (crear y actualizar un costo de prueba). Rama `feature/costos-transporte-tienda` subida; **PR pendiente**.

Documentación: [Vista](../frontend/views/director-de-operaciones/costostransportetienda/costostransportetienda.md) · [Cobertura](../frontend/views/director-de-operaciones/costostransportetienda/components/cobertura-costos.md) · [Reglas del modal de crear](../frontend/views/director-de-operaciones/costostransportetienda/helpers/crear-costo-rules.md).

### Nuevas funcionalidades
- **Pestaña «Cobertura por tienda»** (solo consulta): qué tiendas tienen sus 8 costos activos (entrega y devolución × Interrapidísimo, Envía, Servientrega y D2E), tarjetas por transportadora, filtros, detalle lateral y revisión de las tarifas genéricas `00000`.
- **Modal de crear** propio: listas de tienda (con buscador) y transportadora, segmentados de tipo de guía y estado, resumen antes de crear y aviso de duplicado.
- **Validaciones** por campo: fletes en pesos enteros entre $1.000 y $1.000.000; porcentajes mayores que 0, máximo 100 y 2 decimales con punto. Bloqueo de teclas que ensucian el dato y limpieza al pegar (`"$ 9.520"` → `9520`, `" 2,5 %"` → `2.5`).
- **Modal de actualizar** mejorado (solo para `CostosTransporteTienda`): secciones, campos marcados como «MODIFICADO» y confirmación con *antes → ahora*.

### Cambios
- Cabecera nueva, tabla propia con selector de columnas y estado en pastilla (antes INACTIVO salía verde).
- Paginación `app-pie-paginacion modo="lista"` con números con saltos (`1 2 3 … 352`); 10 registros en computador y 5 en celular, sin scroll interno.
- Diseño para celular (tarjetas, filtros en hoja a pantalla completa, modales a pantalla completa).
- Adaptada al tema **«La Red»**: colores con tokens `--lg-*`, se ve bien en modo oscuro y claro.
- Carpetas como Novedades: `components/`, `helpers/`, `models/`, `styles/`.

### Correcciones
- El modal de actualizar no dejaba escribir (los grupos se recalculaban en cada tecla).
- Scroll con pocas filas en Cobertura (el aviso oculto de la cabecera estiraba la tabla).
- Al buscar o filtrar ya no se mueven los controles.

### Sin cambios
- La lógica de envío y consulta: `onCreateOrModify`, `checkForDuplicateTienda`, `createNewCost`, `fetchTableData` y `updateCampos`. Mismos campos, tipos y colecciones.

### Pruebas
- 57 pruebas unitarias en verde: reglas del modal de crear, formatos, cobertura, componente, `pie-paginacion` y `actualizacion-campos`.

### Pendientes
- Mensaje «Pago creado exitosamente» al crear y `fetchTableData()` doble (lógica vieja).
- Datos: 5 tarifas genéricas incompletas y costos de D2E por tienda.
- PR de `feature/costos-transporte-tienda`.

### Documentación
- Vista, componente de Cobertura, helpers (`crear-costo.rules`, `cobertura.helper`, `formato.helper`) y modelos.
