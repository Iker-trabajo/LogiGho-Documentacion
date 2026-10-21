# Inventario nuevo — pantallas del front

Pantallas de Angular que usan el inventario nuevo (`ApiLambdaInventario`). Estado: **rama `feature/inventario-trazabilidad-front`, validada en PreProd por Iker**; producción pendiente (ver `LogiGho-Dominio-Inventario/11-front/CHECKLIST-PASO-A-PRODUCCION.md`).

Fórmulas del libro (v2): `Actual = Ingresos − Salidas + Anulaciones + Ajustes` · `Consumo = Salidas − Anulaciones` · `En tránsito = Consumo − Entregas − Devoluciones − Pérdidas`.

## Dónde está cada cosa

| Pantalla / pieza | Ruta de código (`SitioLogiGho/src/app/…`) | Quién la ve |
|---|---|---|
| Resumen de inventario | `views/logistica/resumen-inventario/` | roles con acceso al Resumen |
| Menú «Más»: ¿Cómo se calcula? / Administrar inventario | mismo componente | Jefe Logistica y Desarrollador (fórmulas); Jefe Logistica, Desarrollador y CEO (administrar) |
| Modal Bajo stock (con filtro negativo / sin stock / bajo su mínimo) | `…/components/bajo-stock-modal/` y `alertas-resumen.ts` | todos los del Resumen |
| Despachos por día (ventana) | `…/components/despachos-modal/` | todos los del Resumen |
| Trazabilidad por producto | `views/logistica/inventario-trazabilidad/` | Jefe Logistica, Desarrollador, CEO (`ROLES_TRAZABILIDAD`) |
| Administrar inventario | `…/resumen-inventario/administrar-inventario/` | Jefe Logistica, Desarrollador, CEO |
| Catálogo (portafolio): etiqueta de stock y botón Trazabilidad | `views/tienda/portafolio/` | Trazabilidad: Jefe Logistica, Desarrollador, CEO |

Componentes compartidos nuevos: `components/despachos-diarios`, `components/selector-fecha`, `components/select-buscable`, `components/explicacion-calculos`, `components/esqueleto`. Lógica sin Angular: `core/domain/inventario/` (`despachos-diarios.ts`, `envio-lotes.ts`), `core/domain/roles.ts`, `core/domain/colores-logigho.ts`. Estilo común «tabla a tarjetas»: `src/scss/_tabla-tarjetas.scss` (mixin `tabla-a-tarjetas`).

## Despachos por día

- Día = día calendario en **hora de Colombia (America/Bogota, UTC-5)**, no UTC.
- Rangos 7, 14, 30 días o personalizado (hasta 92 días).
- **Fuente temporal:** lectura de la colección `SalidaInventario` por la API genérica (`id producto` exacto + `fechasFiltro` sobre la fecha de creación del documento, hasta 20 páginas de 256; si se alcanza el tope avisa que el resultado puede estar incompleto).
- **Definición pendiente con la arquitecta:** hoy «despachado» = salida registrada en el libro; no es la `Fecha Despacho` de PedidosInter. Un endpoint agregado por día reemplazaría solo `DespachosDiariosService`.
- No verificado: volumen real en producción ni que `fechasFiltro` (por fecha de creación) cubra salidas creadas un día distinto al de su `fecha`.

## Administrar inventario

- Escritura por `metodoGenerico` (booleanos `Tienda.ServicioInventario`, `Productos.ManejaInventario`, `FechaInicioInventario` como texto ISO UTC) + `actualizaInventarioLotes` para el ingreso inicial.
- Regla: nunca un producto activo con la tienda apagada; apagar la tienda desactiva primero sus productos.
- Activar uno o **varios** productos pide el conteo físico inicial (≥ 1) de cada uno; los ingresos **no se reintentan** (no son idempotentes).
- Pendiente de confirmar con la arquitecta: que `metodoGenerico` sea el mecanismo aceptado para esas escrituras.

## Respuestas del API de inventario

El front **solo usa el código HTTP** de las rutas de inventario (`actualizaInventarioLotes`, etc.); no depende del texto de la respuesta (`Operación exitosa.` / `Sin cambios: N pedidos leídos…`). La única lectura de texto está en la creación de guía con la transportadora (`resultado-creacion-guia.ts`), que no es una ruta de inventario.

## Auditoría (`insertarAuditoria`)

`ConsumoGenericoService.insertarGenerico` registra una auditoría con `?coleccion=<nombre del método>`. Para `actualizaInventarioLotes` eso enviaba `coleccion=actualizaInventarioLotes`, que no es una colección: **hipótesis** de la causa de las dos llamadas `insertarAuditoria` con 404 vistas en la consola de PreProd. Cambio hecho: las rutas listadas en `RUTAS_SIN_AUDITORIA` no auditan. **No verificado contra el API** (falta ver que el 404 desaparezca en PreProd).

## Pruebas

Jasmine/Karma (`ng test`). Las suites de inventario pasan; los ~90 fallos restantes de la suite completa son «should create» de componentes y servicios preexistentes, ninguno de inventario.
