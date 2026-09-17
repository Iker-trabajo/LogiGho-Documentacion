## Autor: Iker Acevedo
Fecha creación: 2026-09-16  
Estado: producción (Motor Legacy) + preprod (Motor Dinámico)

---
## Lambda: ApiLambdaLiquidacionesLogighoAOT

**Trigger:** Producción — API Gateway (Cognito). PreProd — Step Function `Financiero-PreProd`, invocación directa.
**AOT:** Sí — `Handler` es el nombre del ejecutable, sin `Ensamblado::Namespace.Clase::Metodo` (formato managed), consistente con publicación nativa (`PublishAot`).

---

## ¿Qué hace?

Liquida guías: calcula cuánto le queda a cada tienda después de flete, servicio, seguro, comisión de recaudo e impuesto de gobierno, y guarda un documento por concepto en `LiquidacionesLogigho`. Puede liquidar todas las guías pendientes de una corrida, o una lista puntual de guías. Ver la [visión general](overview.md) para cómo convive el motor Legacy con el motor Dinámico dentro de esta misma función.

---

## Accionador

| Entorno | Método | Cómo se invoca | Auth |
| --- | ------ | ---------------- | ------------ |
| Producción | — | API Gateway → Lambda, body con `TokenCognito` | Bearer token (Cognito) |
| PreProd | — | Step Function `Financiero-PreProd` → `lambda:invoke` directo | Ninguna (ver [Orquestación PreProd](orquestacion-financiero.md) — ni el motor Dinámico ni la actualización de inventario la necesitan para esta validación) |

---

## Request

```json
{
  "Guias": ["700000001", "700000002"],
  "TokenCognito": "..."
}
```

| Campo | Tipo | Requerido | Descripción |
| ------- | -------- | --------- | ----------- |
| `Guias` | `string[]` | No | Lista puntual de números de guía a liquidar. Vacío o ausente = liquida **todas** las guías pendientes. |
| `TokenCognito` | `string` | Solo en producción | Token del usuario que dispara la liquidación manual desde el front. En PreProd se omite — ver `orquestacion-financiero.md`. |

En PreProd, la Step Function pasa el `Input` de la ejecución tal cual al `body` (`States.JsonToString($)`) — lo que se escriba en "Start execution" en la consola de AWS es exactamente este JSON.

---

## Response

### Exitoso

```json
{
  "DocumentosLiquidados": 42,
  "NoLiquidadas": []
}
```

### Errores

| Código | Cuándo |
| ------ | --------------- |
| `500` | Error interno (Mongo caído, configuración inconsistente, excepción no controlada) |

---

## Flujo interno

```
Function.cs
  -> obtiene ConfiguracionLiquidacionTransportadora, TarifasTransportadoraTienda, TrayectosTransportadora
  -> MapeadorLiquidacionDinamica (Mongo -> modelos de dominio)
  -> LiquidarPedidosDinamicoUseCase.Particionar(pedidos, configuraciones)
       -> lote Dinamicos  -> LiquidarPedidosDinamicoUseCase.LiquidarAsync -> LiquidacionesLogigho
       -> lote Legacy     -> DepurarLiquidacionUseCase.DepuraLiquidacionInter -> LiquidacionesLogigho
```

Los dos lotes corren en la **misma invocación** — no hay exclusión mutua entre motores. Ver [Motor Dinámico → Orquestador](motor-dinamico/orquestador.md) para el detalle de `Particionar`.

---

## Dependencias externas

| Servicio | Uso |
| -------- | ----------- |
| **MongoDB** | Escritura: `LiquidacionesLogigho` (resultado liquidaciones). Lectura: `PedidosInter` (guías), `ConfiguracionLiquidacionTransportadora`, `TarifasTransportadoraTienda`, `TrayectosTransportadora`, `AdministracionCostos`, `Tienda`. |
| **Generico.consumoGenerico** (`URL_SERVICIO_AWS`) | Solo Motor Legacy no-TCC: actualización inventario + lectura colecciones vía API HTTP. Motor Dinámico: Mongo directo, no lo necesita. Omitir esta URL en ejecutables solo-TCC. |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-09-16 | Iker Acevedo | Documentación inicial del contrato de la Lambda (ambos motores) y del despliegue en PreProd. |

---

## Notas importantes

**"AOT" es histórico:** nombre identifica Lambda, pero **no** significa que todo código corra sin reflexión (serialización JSON via Newtonsoft usa reflection).

**`URL_SERVICIO_AWS` es obligatoria en PreProd:** agregada tras primer error real. Pedidos no-TCC fallan sin ella (`"An invalid request URI was provided... BaseAddress must be set"`), porque `Generico.consumoGenerico` la usa como `HttpClient.BaseAddress`. Ocurre solo al mezclar Legacy + Motor Dinámico en misma ejecución. Solución: pasar variable de entorno cifrada en template.
