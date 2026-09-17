## Autor: Iker Acevedo
Fecha creación: 2026-09-16

Estado: producción

# Motor Legacy — `DepurarLiquidacionUseCase`

Motor de liquidación original, en producción sin cambios. Procesa hoy: **Interrapidísimo, Envía, Servientrega**. Cualquier transportadora sin configuración en `ConfiguracionLiquidacionTransportadora` cae aquí por defecto. Ver [Visión general § Ruteo](overview.md#cómo-se-decide-qué-motor-usa-cada-pedido).

**Estructura actual:**
- Archivo único: `DepurarLiquidacionUseCase.cs` (565 líneas).
- Método monolítico: `DepuraLiquidacionInter` (~350 líneas).
- Toda lógica concentrada en un solo lugar.

La complejidad de mantener un método gigante con toda la lógica de negocio es exactamente lo que motivó construir el [Motor Dinámico](motor-dinamico/overview.md) aparte, sin tocar Legacy.

---

## Flujo interno

```mermaid
--8<-- "backend/lambdas-dotnet/lambdas/ApiLambdaLiquidacionesLogighoAOT/diagramas/flujo-legacy.mmd"
```

> Fuente: **[carpeta de diagramas](diagramas/README.md)**.

---

## Cómo entran los datos

El método recibe `List<JArray> lstRequest`, siete arreglos desempaquetados **por posición**, no por nombre:

| Índice | Contenido |
|---|---|
| 0 | Costos operativos por tienda |
| 1 | Productos (para dropshipper) |
| 2 | Tiendas |
| 3 | Costos de transporte por tienda (tarifas legacy) |
| 4 | Guías ya referidas |
| 5 | Marketing |
| 6 | El `Request`: los pedidos a liquidar |

Un `foreach` recorre **todos** los pedidos en un solo ciclo, sin separarlos antes por transportadora ni por tipo — la clasificación Entrega/Devolución es un `if/else if` inline (líneas 79-87).

---

## Cómo decide el flete

**Problema 1: Trayecto hardcodeado**
- Switch textual de nombres (`Urbano`, `Regional`, `Nacional`, `Metropolitano`, `Municipal`, `Reexpedido`).
- Si pedido trae trayecto no reconocido → cae **silenciosamente** a `FleteNacionalMetropolitano`.
- Cero auditoría. Cero forma de saber si eso fue error o fallback.
- Motor Dinámico: valida explícitamente (ver [`ValidadorTrayecto`](motor-dinamico/validacion.md)).

**Problema 2: Búsqueda tarifa en cascada**
- Múltiples `FirstOrDefault()` anidados: tienda+transportadora+estado → sin estado → genérico.
- Cada nivel fallback = fuente de "¿por qué esta tarifa y no otra?".
- Difícil de depurar sin seguir paso a paso.

---

## Construcción de documentos: 4 conceptos, todo mezclado en un método

Legacy crea los mismos 4 tipos de documento que Motor Dinámico, pero sin separar responsabilidades. Todo inline en `if` Entrega (líneas 208-394):

| Documento | Legacy | Motor Dinámico |
|---|---|---|
| **Petición tienda (Entrega)** | Inline en bloque principal | `ConstructorPeticion.ParaEntrega` |
| **Petición proveedor (dropshipper)** | Clona con `JsonConvert` + resetea ~20 campos a mano | `ConstructorPeticion.ParaProveedorDropshipper` + modelo limpio |
| **Impuesto Gobierno** | Bloque aparte (líneas 380-392) | `ConstructorPeticion.ParaImpuestoGobierno` |
| **Devolución** | `if` independiente (NO `else if`); **muta misma instancia** `peticion` | `ConstructorPeticion.ParaDevolucion` siempre nueva instancia |

**Problema real:** Devolución es `if`, no `else if`. Resultado: **pedido con ambas fechas (Entrega + Devolución) genera ambos documentos** para la misma guía.
Motor Dinámico: `ClasificadorTipoLiquidacion` devuelve tipo único o ninguno (sin ambigüedad).

---

## Marketing/Referido: cálculo + I/O juntos

`LiquidacionAdicionalLogigho()` se invoca **dentro del loop** (líneas 255, 267, 346) e **inserta a Mongo inmediatamente**. Cálculo y persistencia mezclados.

Motor Dinámico: separa en dos fases:
1. **Decidir** si pagar: `CalculadoraMarketing`/`CalculadoraReferido` (dominio puro, sin I/O).
2. **Ejecutar** pago (capa aplicación — aún incompleto, ver [fases proyecto](motor-dinamico/orquestador.md#fases-del-proyecto)).

---

## Cómo persiste

En producción (`#else`), inserta llamando a un endpoint HTTP (`/metodoGenerico?coleccion=LiquidacionesLogigho`) — el mismo patrón `consumoGenerico` que usa el resto de la plataforma legacy. Solo en `#if DEBUG` (pruebas locales) escribe directo a Mongo con `IDocumentRepository.InsertarLiquidacionesLogighoAsync`, para no depender de tener el API completo corriendo en la máquina del desarrollador.

La inserción real se hace por lotes de 1000 documentos, con hasta 3 requests concurrentes y reintento con backoff exponencial (Polly).

---

## Por qué no se tocó nada de esto

Justamente porque funciona y mueve plata real hoy. El riesgo de un refactor "mientras tanto" — cambiar algo de este método para que sea más legible o más fácil de extender — es que cualquier diferencia de comportamiento, por mínima que sea, se nota en la próxima liquidación real. La estrategia elegida fue no tocarlo y construir el reemplazo aparte, migrando transportadora por transportadora solo cuando el motor nuevo ya demuestra (con tests y con guías reales comparadas a mano) que calcula exactamente lo mismo. El razonamiento completo está en el [ADR-001](adr/ADR-001-strangler-fig-motor-dinamico.md).

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-09-16 | Iker Acevedo | Documentación inicial — sin cambios de código, este archivo describe el comportamiento tal cual está en producción. |
