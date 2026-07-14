## Autor: Iker Acevedo
Fecha creacion: 2026-07-13

Estado: produccion

## Lambda: ApiLambdaCalcularVentanaTiempo

**Accionador:** Step Functions (primer estado — `Task`)

**AOT:** No

**Posición en el pipeline:** 1 de 5

---

## ¿Qué hace?

Calcula la **ventana de tiempo** (`since` / `until`) que el resto del pipeline usará para consultar Pancake. Es un **cálculo puro**: no toca ninguna base de datos ni API externa. Recibe el tipo de corrida y devuelve el rango en formato **Unix UTC**, más la fecha de reporte y metadatos que se propagan al resto de estados.

La clave del cálculo: la matemática de días se hace en **hora Colombia** (`America/Bogota`, UTC-5) y luego se convierte a Unix — así se evita el bug clásico de calcular "ayer" o "hoy" en UTC y desfasarse 5 horas.

---

## Request

Es lo que envía EventBridge / el Step Function como input de la ejecución.

```json
{
  "slot_id": "1",
  "tipo_calculo": "cierre_dia_anterior"
}
```

| Campo | Tipo | Requerido | Descripción |
| ----- | ---- | --------- | ----------- |
| `slot_id` | `string` | Sí | Identificador de la franja (`"1"`..`"4"`). Solo se propaga como etiqueta. |
| `tipo_calculo` | `string` | Sí | `cierre_dia_anterior` o `intradia_actual`. Determina el rango. |

### Tipos de cálculo

| `tipo_calculo` | `since` (desde) | `until` (hasta) | Uso |
| -------------- | --------------- | --------------- | --- |
| `cierre_dia_anterior` | Ayer 00:00 (Colombia) | Hoy 00:00 (Colombia) | Consolidar el día anterior completo (corrida de las 7am) |
| `intradia_actual` | Hoy 00:00 (Colombia) | Ahora | Foto del día en curso (corridas 9am / 2pm / 5pm) |

---

## Response

```json
{
  "since": 1783832400,
  "until": 1783918800,
  "fecha_reporte": "2026-07-12",
  "tipo": "cierre_dia_anterior",
  "slot_id": "1"
}
```

| Campo | Tipo | Descripción |
| ----- | ---- | ----------- |
| `since` | `long` | Inicio de la ventana en **Unix UTC** |
| `until` | `long` | Fin de la ventana en **Unix UTC** |
| `fecha_reporte` | `string` | Fecha que representa el reporte (`yyyy-MM-dd`) |
| `tipo` | `string` | Eco del `tipo_calculo` recibido |
| `slot_id` | `string` | Eco del `slot_id` recibido |

> En el Step Function, esta salida se guarda en `$.ventana` y luego se **inyecta a cada página** en el 2º Map vía `ItemSelector`.

---

## Flujo interno

```
FunctionHandler (Function.cs)
  -> CalcularVentanaUseCase.Ejecutar(entrada)
       -> Obtiene "ahora" desde TimeProvider (inyectado, testeable)
       -> Convierte a hora Colombia (America/Bogota, UTC-5)
       -> Según tipo_calculo:
            cierre_dia_anterior -> [ayer 00:00, hoy 00:00)
            intradia_actual     -> [hoy 00:00,  ahora)
       -> Convierte los límites (hora Colombia) a Unix UTC
       -> Arma fecha_reporte
  -> [RESULT] { since, until, fecha_reporte, tipo, slot_id }
```

---

## Arquitectura Clean Architecture

```
ApiLambdaCalcularVentanaTiempo/
├── Function.cs                       ← Entry point (Step Functions)
├── Aplicacion/
│   ├── CasosUso/
│   │   └── CalcularVentanaUseCase.cs ← Lógica pura del cálculo
│   └── DTO/
│       ├── EntradaVentana.cs         ← { slot_id, tipo_calculo }
│       └── SalidaVentana.cs          ← { since, until, fecha_reporte, ... }
└── Dominio/
    └── (sin repositorios — cálculo puro)
```

> No tiene capa de Infraestructura: no accede a Mongo ni a ninguna API. Es la lambda más simple y la más fácil de testear.

---

## Variables de entorno

| Variable | Descripción |
| -------- | ----------- |
| — | Ninguna. Es un cálculo puro. |

---

## Configuración Lambda

| Parámetro | Valor |
| --------- | ----- |
| Runtime | `dotnet8` |
| Handler | `ApiLambdaCalcularVentanaTiempo::ApiLambdaCalcularVentanaTiempo.Function::FunctionHandler` |
| Memory | `256 MB` |
| Timeout | `15 segundos` |
| Architecture | `x86_64` |

---

## Historial de cambios

| Fecha | Autor | Cambio |
| ----- | ----- | ------ |
| 2026-07-13 | Iker Acevedo | Creación. Cálculo de ventana con `TimeProvider` inyectable y matemática de días en `America/Bogota`. |

---

## Observaciones

- **`TimeProvider`** se inyecta para poder testear el cálculo con un reloj falso (fechas fijas) sin depender de la hora real.
- El `slot_id` no participa en el cálculo — solo se propaga como etiqueta para identificar la franja en la salida final (queda guardado en cada registro de estadística).
- Si se necesitara una nueva ventana (ej. "últimos 7 días"), se agrega un nuevo `tipo_calculo` en el caso de uso, sin tocar el resto del pipeline.
