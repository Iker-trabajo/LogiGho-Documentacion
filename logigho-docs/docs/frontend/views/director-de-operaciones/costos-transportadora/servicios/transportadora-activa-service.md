## Autor: Iker Acevedo
Fecha creación: 2026-09-16
Estado: preprod
Tipo: servicio

# `TransportadoraActivaService`

Estado compartido de la transportadora seleccionada en el Hub — provisto a nivel del componente `CostosTransportadoraComponent` (no `providedIn: 'root'`), así que cada vez que se entra al módulo arranca limpio. Las 5 pestañas leen de este servicio en vez de recibir la data por `@Input()` en cascada.

Usa **Angular Signals** para reactividad (`configuracion`, `trayectos`, `tarifas`, `excepciones`, `tiendas`, `cargando`) — sin RxJS, salvo en las llamadas HTTP en sí, que siguen siendo Observables/Promises como el resto del CRUD genérico de la plataforma.

---

## Mapeo camelCase (TS) ↔ PascalCase (Mongo)

Las colecciones reales guardan sus campos en PascalCase (`IdTransportadora`, `MotorLiquidacion`...) — la convención de todo `LogighoDB`. El resto del front usa camelCase en todos sus modelos. En vez de contaminar cada componente con nombres PascalCase, la conversión pasa **una sola vez**, acá, en el borde donde se habla con Mongo — los componentes nunca ven un campo en PascalCase.

Cuatro pares de funciones puras hacen la conversión: `configuracionAMongo`/`configuracionDesdeMongo`, `trayectoAMongo`/`trayectoDesdeMongo`, `tarifaAMongo`/`tarifaDesdeMongo`.

---

## Acciones principales

| Método | Qué hace |
|---|---|
| `seleccionar(card)` | Guarda la transportadora activa y dispara 5 cargas en paralelo (`Promise.all`): configuración, trayectos, tarifas, auditoría, catálogo de tiendas |
| `actualizarConfiguracion(config)` | Upsert por `_id` en `ConfiguracionLiquidacionTransportadora`, y refresca el signal con lo que confirmó Mongo |
| `guardarTrayecto` / `eliminarTrayecto` | Upsert / delete en `TrayectosTransportadora` |
| `marcarComoDefecto(codigo)` | Actualiza `ConfiguracionTransportadora.trayectoPorDefecto` — **no** toca el documento del trayecto (ver comentario en el código: mezclar "es default" ahí duplicaría la fuente de verdad) |
| `guardarTarifa` / `eliminarTarifa` | Upsert / delete en `TarifasTransportadoraTienda` |

---

## Catálogo de tiendas: cacheado, no por transportadora

`cargarCatalogoTiendas()` no vuelve a pedir el catálogo si ya hay tiendas cargadas (`if (this.tiendas().length > 0) return`) — el catálogo de tiendas activas no depende de qué transportadora esté seleccionada, así que no tiene sentido repetir la consulta al cambiar de tab o de transportadora dentro de la misma sesión del componente.

Un detalle no obvio: el identificador de un documento de `Tienda` es el campo corto **`Id`**, no `IdTienda` — ese nombre más largo se usa como llave foránea en *otras* colecciones (por ejemplo `TarifasTransportadoraTienda.IdTienda`), pero el documento de Tienda en sí lo guarda como `Id`. Confirmado contra `administracion/tiendas/tiendas.component.ts`, que ya lee y escribe esa colección en producción.

---

## Computed values útiles

- `esMotorDinamico` — `true` si `configuracion().motorLiquidacion === Dinamica`.
- `usaTablaIncrementos` — `true` si la política de peso es `TablaIncrementosConTope`; condiciona qué columnas se muestran en Tarifas por Tienda.
- `topeKilos` — el tope configurado, o `5` por defecto si todavía no hay configuración cargada.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-09-16 | Iker Acevedo | Documentación inicial del servicio. |
