## Autor: Iker Acevedo

Fecha creacion: 2026-11-06  
Estado: desarrollo (PreProd validada 2026-11-06; producción pendiente)

---

## Lambda: ApiLambdaNovedadesInterDistancias

**Trigger:** Step Function (`Step_Procesos_Logigho`), rama propia del `Parallel`  
**AOT:** No (runtime administrado `dotnet10`, 512 MB, timeout 300 s)  
**Historia:** 027 · **Spec:** 012 · **Proyecto:** el mismo de [ApiLambdaNovedadesInter](ApiLambdaNovedadesInter.md) (un zip, tres funciones; ver [ADR-003](adr/ADR-003-distancias-y-reportes-en-el-mismo-proyecto.md))

> Convención de marcas: **observado** (ruta o id citado), **Iker** (fecha) o **hipótesis**. Lo que depende de producción
> está marcado **pendiente**: producción aún no tiene estas funciones.

---

## ¿Qué hace?

Calcula a cuántos metros **por calle, en carro**, está el punto donde el mensajero de Inter marcó cada intento de la dirección
del pedido, y lo deja en `SeguimientoNovedadesInter` (metros, color, estado y enlace a la ruta). Sirve para que la gerencia
reclame a Inter las novedades marcadas lejos de la dirección y para que la vista de novedades muestre la distancia
(decisión de Iker 2026-11-05).

Las reglas de negocio (confianza de la geocodificación, colores, confirmación del rojo, estados) están en
[Reglas de la distancia](dominio/reglas-distancia.md). Esta página describe la función.

---

## Accionador

| Origen | Entrada | Auth |
| ------ | ------- | ---- |
| `Step_Procesos_Logigho` (rama nueva del `Parallel`) | `{"origen":"step"}` (invocación directa, **no** formato API Gateway) | IAM (rol de la Step) |
| A mano (soporte) | `{}` o `{"modo":"backfill"}` | IAM |

- Handler: `ApiLambdaNovedadesInter::ApiLambdaNovedadesInter.Handlers.DistanciasHandler::EjecutarAsync` (observado: `aws-lambda-distancias.json`).
- `modo = backfill` **no cambia el algoritmo**: solo queda anotado en el log. La selección ya cubre todo lo auditado, por tandas
  (observado: `DistanciasHandler.cs`). Cualquier otro valor se registra como `modo-desconocido` y se trata como normal.
- La Step corre cada 3 h. La rama de Distancias va **en paralelo** a la de solicitar novedades y no espera a nadie: encuentra
  completos los callbacks del ciclo anterior, así que el retraso máximo de una novedad es de 3 h (observado en CloudWatch prod
  2026-11-05; fuente: `specs/012_novedades_distancia_reportes/spec.md` §2.1 del workspace).

---

## Request

```json
{ "origen": "step" }
```

| Campo | Tipo | Requerido | Descripción |
| ----- | ---- | --------- | ----------- |
| `origen` | `string` | No | Solo informativo (`step`, `directo`); queda en el log |
| `modo` | `string` | No | `backfill` o vacío. Solo informativo |

---

## Response

### Exitoso (resumen de la corrida)

La función devuelve el resumen en JSON y emite las métricas EMF de la sección [Métricas](#metricas).

```json
{
  "CorridaId": "<id de la invocación>",
  "CandadoOcupado": false,
  "Interrumpida": false,
  "CorridaCortada": false,
  "GuiasSeleccionadas": 120,
  "GuiasProcesadas": 118,
  "GuiasConError": 0,
  "GuiasOmitidas": 2,
  "Geocodificaciones": 40,
  "Rutas": 95,
  "Confirmaciones": 12,
  "LlamadasGoogle": 147,
  "PendientesPorTope": 0,
  "PendientesPorTiempo": 0
}
```

(Valores de ejemplo inventados; la forma sale de `ResumenesFunciones.cs`.)

| Campo | Significado |
| ----- | ----------- |
| `CandadoOcupado` | Otra corrida vigente tiene el candado: esta no hizo nada |
| `Interrumpida` | Se acabó el presupuesto de 240 s o el tope de la lambda: lo pendiente queda para la siguiente corrida |
| `CorridaCortada` | `FALLAS_SEGUIDAS_PARA_CORTAR` fallas seguidas de Google (o de escrituras): se cortó y se avisó por Slack |
| `GuiasOmitidas` | Siguen sin datos del pedido (sin pedido o sin dirección) y nada cambió: no se tocan ni se llama a Google |
| `LlamadasGoogle` | `Geocodificaciones + Rutas + Confirmaciones`. En una segunda corrida sin cambios debe ser **0** (CA5) |
| `PendientesPorTope` / `PendientesPorTiempo` | Guías que quedaron para la siguiente corrida por el tope o por el tiempo |

### Errores

| Situación | Qué pasa |
| --------- | -------- |
| Google responde 429, 5xx o hay timeout | Reintenta 2 veces (esperas de 1 s y 3 s) |
| Google responde 400 o 404 | Resultado definitivo de **esa** petición (dirección o ruta imposible); no es falla del servicio |
| 401/403 o `REQUEST_DENIED` | Error de servicio (llave o permisos): sin reintento; cuenta como falla seguida |
| `FALLAS_SEGUIDAS_PARA_CORTAR` fallas seguidas | Corta la corrida y manda alerta a Slack («Google Maps está fallando: se cortó la corrida de distancias») |
| Error en una guía | Se registra `guia-con-error` y la corrida sigue; cuenta como falla seguida |
| Excepción no controlada | Se registra `excepcion-no-controlada` y se **relanza**: el `Catch` de la rama la absorbe y las demás ramas de la Step terminan igual (CA6) |

Observado en: `CalcularDistanciasUseCase.cs`, `ClienteGoogleMaps.cs`, `DistanciasHandler.cs`.

---

## Flujo interno

```
Step_Procesos_Logigho
  Parallel
    rama 1: Lambda Novedades Inter (solicitar) …
    rama 2: Lambda Distancias Novedades  ── {"origen":"step"} ──> DistanciasHandler
         -> CalcularDistanciasUseCase
              1. ICandado: toma "distancias" en CandadosNovedadesInter (TTL 15 min); si está ocupado, termina
              2. IRepositorioDistancias.ObtenerPendientesAsync(tope)
                   (HuellaMotivos con valor y HuellaDistancias distinta de "{HuellaMotivos}|v3";
                    activos primero, luego FechaActualizacion descendente)
              3. ObtenerDireccionesAsync: PedidosInter por Numeropreenvio (Direccion, Ciudad, Departamento) — solo lectura
              4. por guía, con concurrencia CONCURRENCIA_GOOGLE y presupuesto de 240 s:
                   a. IGeocodificador (Google Geocoding) solo si nunca se geocodificó o cambió la huella de la dirección
                   b. IRuteador (Google Routes, DRIVE) una vez por punto nuevo (caché por HuellaGeo)
                   c. si alguna ruta da rojo: UNA geocodificación de confirmación con la dirección completa
                   d. GuardarAsync: $set parcial de DireccionPedido, Distancias, DistanciaUltima, HuellaDistancias
              5. libera el candado y emite log [RESULT] + métricas
```

- La Step de **PreProd** la maneja Terraform (`environments/preprod/procesos-automaticos`): el `Parallel` se llama
  `Procesos en paralelo` y la rama es `Lambda Distancias Novedades` → `Fin Distancias`, con `TimeoutSeconds: 330`, `Retry`
  solo para errores transitorios de Lambda y `Catch` de `States.ALL` hacia el fin (observado: `step_procesos_logigho.asl.json.tftpl`).
- La Step de **producción** no está en Terraform: la rama se agrega con
  `pipelines/novedades/scripts/agregar_distancias_a_step_function.py` (`ensayo`, `aplicar`, `revertir`), que añade una rama al
  `Parallel Init` y una política IAM que permite invocar **solo** esta función. **Pendiente de ejecutar** (lo hace Iker).

### Candado

- Colección `CandadosNovedadesInter`, documento `{ _id: "distancias", Dueno, ExpiraEn }`.
- Se toma con un `upsert` atómico cuyo filtro solo coincide si el candado no existe o ya venció; si está vigente, Mongo responde
  clave duplicada y la función entiende «ocupado». El vencimiento se comprueba en el código, así que funciona aunque el índice TTL
  tarde en limpiar (observado: `CandadoMongo.cs`). Duración: 15 min (`ConfiguracionDistancias.DuracionCandado`).

### Tope por corrida y presupuesto de tiempo

| Parámetro | Valor | Origen |
| --------- | ----- | ------ |
| `MAX_GUIAS_DISTANCIA_POR_CORRIDA` | 1500 por defecto | variable de entorno |
| Presupuesto de tiempo | 240 s fijos | `ConfiguracionDistancias` |
| `CONCURRENCIA_GOOGLE` | 4 por defecto | variable de entorno |
| `FALLAS_SEGUIDAS_PARA_CORTAR` | 5 por defecto | variable de entorno |
| Tope de la función | 300 s; se reservan 10 s antes del tope para cerrar limpio | `aws-lambda-distancias.json`, `DistanciasHandler.cs` |

Lo que no alcanza queda pendiente y se retoma en la siguiente corrida (cada 3 h). El **relleno** de lo ya auditado no es otro
proceso: son las mismas corridas hasta vaciar los pendientes. El relleno de producción cubre **todas** las novedades, sin límite
(Iker 2026-11-06).

---

## Variables de entorno (solo nombres)

| Variable | Origen | ¿Cifrada? | Descripción |
| -------- | ------ | --------- | ----------- |
| `ENTORNO` | Terraform | No | `preprod` o `prod` (etiqueta de métricas y alertas) |
| `DATABASE_NAME` | Terraform | No | Base Mongo |
| `CADENA_CONEXION` | Pipeline | Sí (AES) | Conexión a Mongo |
| `GOOGLE_MAPS_API_KEY` | Pipeline | Sí (AES) | Llave de **servidor** de Google Maps Platform (Geocoding + Routes) |
| `MAX_GUIAS_DISTANCIA_POR_CORRIDA` | Opcional | No | Tope de guías por corrida (1500) |
| `CONCURRENCIA_GOOGLE` | Opcional | No | Guías en paralelo (4) |
| `FALLAS_SEGUIDAS_PARA_CORTAR` | Opcional | No | Fallas seguidas para cortar la corrida (5) |
| `SLACK_WEBHOOK_ALERTAS` | Pipeline | Sí (AES) | Alertas de la corrida cortada (opcional: sin él solo queda el log) |

Detalle de cada entorno en [Variables de entorno](../../../../../guias/variables-entorno.md).

---

## Dependencias externas

| Servicio | Uso |
| -------- | --- |
| `MongoDB` (Atlas en PreProd, DocumentDB en producción) | Lee `SeguimientoNovedadesInter` y `PedidosInter` (solo lectura); escribe `SeguimientoNovedadesInter` (`$set` parcial) y `CandadosNovedadesInter` |
| `Google Geocoding API` | `GET https://maps.googleapis.com/maps/api/geocode/json` con `components=country:CO`, `region=co`, `language=es` |
| `Google Routes API` | `POST https://routes.googleapis.com/directions/v2:computeRoutes`, `travelMode = DRIVE`, `routingPreference = TRAFFIC_UNAWARE`, `X-Goog-FieldMask = routes.distanceMeters,routes.duration` |
| `Slack` (webhook entrante) | Alerta de corrida cortada |
| `CloudWatch` | Logs, métricas EMF y alarmas |

En producción la función corre en la misma VPC, subred y grupo de seguridad que `ApiLambdaNovedadesInter` (DocumentDB privado y
salida por NAT hacia Google y Slack); en PreProd no necesita VPC (Atlas público). Observado: `environments/prod/novedades/main.tf`.

---

## Métricas

Namespace `Logigho/NovedadesInter`, dimensiones `Entorno` y `Operacion = distancias` (observado: `DistanciasHandler.cs`):

`GuiasSeleccionadas`, `GuiasProcesadas`, `GuiasConError`, `Geocodificaciones`, `Rutas`, `LlamadasGoogle`, `PendientesPorTope`,
`PendientesPorTiempo`, `CandadoOcupado`, `CorridaCortada`, `CorridaInterrumpida`.

Las alarmas están en [Observabilidad](infraestructura/observabilidad.md).

### Logs

Una línea por evento, con `corridaId` y contadores: `[REQUEST]` al inicio y en `pendientes`, `[RESULT]` en `fin`
(`seleccionadas`, `procesadas`, `conError`, `omitidas`, `geocodificaciones`, `rutas`, `confirmaciones`, `llamadasGoogle`,
`pendientesPorTope`, `pendientesPorTiempo`, `interrumpida`, `cortada`, `duracionMs`).

**Nunca** se loguea la dirección, el nombre, el teléfono, la llave de Google ni el token; solo guía, conteos, estados y
códigos HTTP (CA13).

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-11-06 | Iker Acevedo | Creación de la función (PR de Lambdas 619, 622, 623, 624) y primera validación en PreProd |

---

## Observaciones

- **Resultado observado en PreProd (2026-11-06, 277 guías copiadas de prod):** 125 calculadas, 0 en `Error`, 38 rojas confirmadas;
  corrida de 17 s y 136 MB de memoria (ver [plan de pruebas](../../../../../guias/novedades-distancia-reportes-pruebas.md)).
  **Pendiente:** el comportamiento sobre DocumentDB de producción (el `$set` y la lectura se prueban primero en solo lectura).
- **Memoria sobredimensionada:** usa ~136 MB de 512. Probar 256 (mejora **M-261**).
- **Cupo del tope:** las guías sin dirección en `PedidosInter` siguen ocupando cupo del tope de 1500 aunque se omiten sin llamar a
  Google (mejora **M-258**).
- **Lectura completa por corrida:** lee una proyección mínima de todos los seguimientos y compara `HuellaDistancias` en memoria
  (en DocumentDB `$expr` se soporta a medias). Vigilar duración cuando la colección pase de ~50 000 (mejora **M-257**).
- **Criterio de ciudad:** la coincidencia de ciudad es por contención en un sentido u otro («CALI» dentro de «CALIMA» pasaría);
  hay que revisarlo (mejora **M-263**).
- **Alarma de corrida cortada:** `CorridaCortada` de Distancias se avisa por Slack y se emite como métrica, pero no existe una
  alarma de CloudWatch propia para esa métrica (las alarmas `corrida-cortada` miran `Operacion = solicitar`). Observado en
  `modules/novedades-inter/alarmas.tf`.
- **Versión de la regla:** el código vigente es **v3** (`VersionReglaGeocodificacion.Actual`); la spec 012 aún dice v2. Ver
  [Reglas de la distancia](dominio/reglas-distancia.md#version-de-la-regla-y-huella).
