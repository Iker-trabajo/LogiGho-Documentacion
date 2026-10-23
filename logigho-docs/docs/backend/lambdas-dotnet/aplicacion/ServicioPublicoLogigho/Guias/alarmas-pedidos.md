# Alarmas de pedidos en Servicios_Guias

> Estado: decisiones aprobadas por Iker (2026-10-22). Lo descrito como "hoy" sale de leer el código; **no se verificó en AWS** si `/validacionPedidos` está desplegada.

## 1. Qué son las alarmas

Controles que revisan si un pedido nuevo se parece a pedidos anteriores del mismo cliente (venta duplicada, historial de entregas y devoluciones). Hoy solo las usa la **carga masiva** (`ApiLambdaCargarGuias`). La creación manual (`ApiLambdaCrearGuia`) no las aplica.

## 2. Cómo funcionan hoy, de punta a punta

```
Front (toggle "Validar Alarmas")
  -> POST cargaPreenvios { IdCarga, TipoEtiqueta, CargarAlarmas }
     -> ApiLambdaCargarGuias/Function.cs:76-79
          si CargarAlarmas == false:
             POST /validacionPedidos { IdCarga }
                -> ApiLambdaValidacionesCarga -> ValidacionesCargaNegocio
                   calcula 5 campos y los guarda en CargaPedido (PUT por fila)
     -> por cada fila: Carga<Transportadora>UseCase.procesarRegistroCarga
          -> procesarAlarmas (solo actúa si CargarAlarmas == false
             y la fila trae el campo "Cliente Existe")
             si salta una regla: Estado="Rechazado", ERROR="<motivo>" y la fila no genera guía
```

### Qué calcula `/validacionPedidos`

Código: `Lambdas_NET_LogiGho/LambdasLogiGho.Dominio/Negocio/CargaInter/ValidacionesCargaNegocio.cs`. Lambda: `ApiLambdaValidacionesCarga/.../Function.cs:63-70`.

| Campo escrito en `CargaPedido` | Cómo se calcula |
|---|---|
| `Cliente Existe` | Hay un pedido anterior en `PedidosInter` con el mismo `Telefono` **o** `Identificacion` (línea 92) |
| `Venta Duplicada 48 Horas` | Igual que arriba, con `FechaCarga` en las últimas 48 h (línea 111) |
| `Venta Duplicada 8 Dias` | Igual, con `FechaCarga` en los últimos 8 días (línea 119) |
| `Porcentaje Entregas` | Pedidos previos con el mismo `Telefono` que tienen `Fecha Entrega`, sobre el total (línea 142) |
| `Porcentaje Devoluciones` | Pretende usar `Fecha Devolucion`, pero hoy repite la cuenta de entregas (ver bug 2) |

- **Colecciones:** lee toda `PedidosInter` y toda `ValidacionPedidos`; actualiza `CargaPedido`; escribe conteos por teléfono en `ValidacionPedidos`.
- **Identificación del cliente:** teléfono para los porcentajes; teléfono o identificación para duplicados y "Cliente Existe".
- **Excluye** los pedidos de la misma carga (`Idcarga` distinto, línea 94). Duplicados dentro de la misma carga los trata otro mecanismo ("Registro Duplicado").
- **Ventanas:** 48 horas y 8 días. La de 8 días incluye a la de 48 h, así que la de 48 h nunca es la única que salta.
- **Mínimo de historial:** no hay. Con un solo pedido previo ya calcula porcentajes. Sin historial da 0 %.
- **Cliente Existe** solo informa; nunca bloquea.

### Qué rechaza `procesarAlarmas`

Copiado en 6 archivos (`CargaInterUseCase.cs:381`, `CargaEnviaUseCase.cs:367`, `CargaD2EUseCase.cs:367`, `CargaXCargoUseCase.cs:307`, `CargaServientregaUseCase.cs:322`, `CargaTccUseCase.cs:403`). Reglas y orden idénticos en las 6:

1. `Venta Duplicada 48 Horas == "Si"` -> rechaza.
2. `Venta Duplicada 8 Dias == "Si"` -> rechaza.
3. `Porcentaje Devoluciones > 50` -> rechaza.
4. `Porcentaje Entregas < 50` y `> 0` -> rechaza.

Si la fila no trae `Cliente Existe` (la validación no corrió, o la fila no tiene teléfono ni identificación), pasa sin revisar nada.

## 3. Ejemplos (con la validación corriendo y `procesarAlarmas` activo)

**A. Pedido normal.** Cliente nuevo, sin pedidos previos. Cliente Existe=No, duplicadas=No, entregas 0 %, devoluciones 0 %. Ninguna regla salta: se genera la guía.

**B. Venta duplicada.** Mismo teléfono con un pedido de otra carga hace 30 horas: 48 h=Si y 8 días=Si. **Rechazado**, `ERROR="Venta Duplicada 48 Horas"`. Si el pedido previo es de hace 5 días, solo 8 días=Si y el motivo es `"Venta Duplicada 8 Dias"`.

**C. Cliente con muchas devoluciones.** Teléfono con 4 pedidos viejos (más de 8 días): 1 entregado, 3 devueltos.
- Esperado: devoluciones 75 %, rechazo por devoluciones altas.
- Hoy: entregas 25 % y devoluciones 25 % (bug 2). Devoluciones 25 no supera 50, pero entregas 25 está entre 0 y 50: **Rechazado** con `"Porcentaje Entregas 25 %"`. Rechaza, pero con el motivo equivocado.
- Efecto colateral: un cliente con 4 de 4 entregados saca 100 % en ambos y se rechaza con `"Porcentaje Devoluciones 100 %"`. Con alarmas activas solo pasan los clientes con historial de 0 % o exactamente 50 %.

## 4. Bugs e incoherencias encontrados

| # | Dónde | Problema |
|---|---|---|
| 1 | `ApiLambdaCargarGuias/.../Function.cs:76`; `procesarAlarmas` en los 6 `Carga*UseCase`; front `step-resumen.component.html:199-204`, `genera-guias.component.ts:316,424` | **Flag invertido.** El backend aplica las alarmas cuando `CargarAlarmas == false`. El front lo manda `true` con el interruptor "Validar Alarmas" encendido (valor por defecto). Con el interruptor encendido no hay alarmas; apagado y confirmando "No generarás las guías con alarmas", sí se aplican. Contradice `ApiLambdaCargarGuias.md:45`. |
| 2 | `ValidacionesCargaNegocio.cs:143` | `Porcentaje Devoluciones` se calcula con `totalConFechaEntrega` en vez de `totalConFechaDevolucion`. Siempre es igual a `Porcentaje Entregas`. |
| 3 | `ValidacionesCargaNegocio.cs:93,112,120` | Posibles falsos duplicados: si la fila trae teléfono pero no identificación, `Identificacion == ""` puede coincidir con pedidos viejos de identificación vacía. Sospecha por lectura de código, no probada con datos. |
| 4 | `ValidacionesCargaNegocio.cs:99` vs `:92,:111,:119` | Los porcentajes usan solo teléfono y los duplicados teléfono o identificación. Sin criterio único. |
| 5 | `ValidacionesCargaNegocio.cs:16,45,151` | Descarga colecciones completas y hace un `PUT` por fila. No sirve para tiempo real. |
| 6 | 6 archivos `Carga*UseCase` | `procesarAlarmas` está copiado; un cambio hay que repetirlo 6 veces. |
| 7 | `ValidacionesCargaNegocio.cs:153-175` | `ValidacionPedidos` se escribe pero ningún código la lee después. |
| 8 | `modal-generar-guias.component.md:117` | La documentación dice que `validacionPedidos` no está en funcionamiento. La lambda existe en código pero no está en Terraform ni documentada. Si no está desplegada, `Cliente Existe` nunca se escribe y las alarmas se saltan siempre. **Pendiente de verificar en AWS.** |
| 9 | `ApiLambdaValidacionesCarga/.../Function.cs:58` | Solo lee el header `Token` con esa capitalización exacta; `ApiLambdaCargarGuias` acepta `Token` y `token`. |

## 5. Decisiones para la API (reglas RA)

Las alarmas **sí se usan** en la API.

| Regla | Decisión |
|---|---|
| **RA-001** | Autorización: al entregar credenciales, LogiGho indica las tiendas disponibles. El `IdTienda` del payload debe pertenecer a esas credenciales; si no, se rechaza antes de evaluar alarmas. |
| **RA-002** | Venta duplicada **rechaza** (HTTP 409, sin crear la guía). Criterio: mismo teléfono **o** misma identificación, ignorando valores vacíos. Ventana única de **8 días** (la de 48 h queda incluida). |
| **RA-003** | Porcentajes de devoluciones y entregas solo **marcan**: la guía se crea y el response informa el dato. No bloquean hasta corregir el cálculo (bug 2) y definir historial mínimo. |
| **RA-004** | El integrador **no puede forzar** la creación cuando salta una alarma que bloquea. Un caso legítimo se resuelve con LogiGho. |
| **RA-005** | Interruptor encender/apagar alarmas: lo controla un **admin de LogiGho** en el panel, **por tienda**, más un interruptor global de emergencia. El integrador no lo cambia por API. |
| **RA-006** | Todo response (guía creada o rechazada) lista las alarmas evaluadas en `alarmas[]`, cada una explicada, y un campo `alarmasActivas` (true/false según el interruptor de la tienda). |
| **RA-007** | La consulta de historial debe ir por índice (teléfono/identificación + fecha). Nunca se lee una colección completa. |

### Forma propuesta del response (RA-006)

Guía creada con una alarma que marca:

```json
{
  "estado": "Creada",
  "alarmasActivas": true,
  "alarmas": [
    { "codigo": "VENTA_DUPLICADA", "descripcion": "Mismo teléfono o identificación con pedido en los últimos 8 días", "valor": 0, "umbral": 1, "resultado": "ok" },
    { "codigo": "PORCENTAJE_DEVOLUCIONES", "descripcion": "Porcentaje de pedidos previos del cliente con devolución", "valor": 60, "umbral": 50, "resultado": "marcada" }
  ]
}
```

Rechazo por venta duplicada (409):

```json
{
  "estado": "Rechazada",
  "alarmasActivas": true,
  "alarmas": [
    { "codigo": "VENTA_DUPLICADA", "descripcion": "Mismo teléfono o identificación con pedido en los últimos 8 días", "valor": 1, "umbral": 1, "resultado": "bloqueo" }
  ]
}
```

`resultado`: `ok` (no salta), `marcada` (informa, no bloquea), `bloqueo` (rechaza). Con alarmas apagadas para la tienda: `alarmasActivas:false` y `alarmas` vacío. Nombres de campos y códigos son propuestos; confirmar al diseñar el contrato.

## 6. Pendientes

- Confirmar en AWS si `/validacionPedidos` está desplegada (bug 8).
- Corregir en la carga masiva el flag invertido (bug 1) y el cálculo de devoluciones (bug 2). No es parte de la API nueva.
- Definir historial mínimo para los porcentajes (RA-003).
- Pantalla de admin y almacenamiento del interruptor por tienda (RA-005).
- Definir los códigos y campos finales del response (RA-006).
