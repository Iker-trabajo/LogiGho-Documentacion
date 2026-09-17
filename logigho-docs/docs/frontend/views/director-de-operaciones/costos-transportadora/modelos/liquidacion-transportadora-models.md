## Autor: Iker Acevedo
Fecha creación: 2026-09-16
Estado: preprod
Tipo: modelo

# Modelos — `liquidacion-transportadora.models.ts`

Interfaces y enums que describen el dominio de configuración de liquidación. Están alineados **1:1** con el dominio real del backend (`Dominio/Modelos/Configuracion/*.cs`, `Dominio/Constantes/Enums.cs`) — no son una interpretación propia del front, son la misma fuente de verdad copiada literal, para que un cambio de un lado nunca deje al otro leyendo algo distinto.

---

## Por qué los enums son idénticos en texto al backend

```ts
export enum MotorLiquidacion {
  Dinamica = 'Dinamica',
  Legacy = 'Legacy',
  Sombra = 'Sombra',
  Deshabilitado = 'Deshabilitado'
}
```

El valor string de cada miembro usa PascalCase, igual al nombre del enum en C#. Cuando la Lambda serializa con `JsonStringEnumConverter` (el enfoque estándar de .NET), el texto que llega al front ya calza sin necesitar una tabla de traducción intermedia — evita el tipo de bug donde el front espera `"dinamica"` en minúscula y el backend manda `"Dinamica"`.

Los 4 enums del dominio (`MotorLiquidacion`, `PoliticaPeso`, `TipoLiquidacion`, `AlcanceTarifa`) y el catálogo de 9 `MotivoNoLiquidacion` siguen el mismo criterio — ver la tabla completa en [Validación (backend)](../../../../../backend/lambdas-dotnet/lambdas/ApiLambdaLiquidacionesLogighoAOT/motor-dinamico/validacion.md#catálogo-completo-de-motivonoliquidacion).

Cada enum tiene su propio `_TEXTO: Record<Enum, string>` — el único punto de traducción a texto legible, para que ninguna pantalla invente su propia etiqueta.

---

## Interfaces principales

| Interfaz | Colección Mongo | Espejo de (backend) |
|---|---|---|
| `ConfiguracionTransportadora` | `ConfiguracionLiquidacionTransportadora` | `ConfiguracionTransportadora.cs` |
| `TrayectoTransportadora` | `TrayectosTransportadora` | `TrayectoTransportadora.cs` |
| `TarifaTransportadoraTienda` | `TarifasTransportadoraTienda` | `TarifaTransportadoraTienda.cs` |
| `AuditoriaExcepcion` | `AuditoriaLiquidaciones` (colección que **todavía no existe** — ver [Simulador de Liquidación](../components/simulador-liquidacion.md)) | `PedidoLiquidable.cs` + `MotivoNoLiquidacion` |
| `TiendaCatalogo` | `Tienda` (colección compartida con el resto de la app) | — |

Un detalle que vale la pena señalar porque no es obvio mirando solo el backend: `porcentajeKiloAdicional` **no** vive en `ConfiguracionTransportadora` — en el dominio real ese dato es por tienda y por trayecto (vive en `TarifaTransportadoraTienda`, y encima es una tabla de incrementos, no un solo porcentaje). Se configura en la pestaña Tarifas por Tienda, no en Configuración General, aunque a primera vista parecería un dato "general".

---

## `MOTIVO_NO_LIQUIDACION_TAB`

Mapa que conecta cada motivo de no-liquidación con la pestaña donde se corrige (`trayectos`, `tarifas`, `configuracion`), o `null` cuando el motivo no es un problema de configuración (es un error operativo puntual — no hay pestaña que lo "arregle"). Lo usa el botón "Ir a configurar" de [Auditoría de Excepciones](../components/auditoria-excepciones.md).

---

## Por qué no hay una interfaz `ResultadoSimulacion`

A propósito. El motor Dinámico calcula dentro del dominio de la Lambda y ese dominio todavía no expone un endpoint HTTP — solo se invoca desde dentro del proceso real de liquidación. Reproducir esa aritmética en TypeScript, aunque fuera "aproximada", significaría tener **dos lugares calculando plata** con la posibilidad real de que se desincronicen — el riesgo que se decidió evitar en todo este módulo. Ver [ADR-001](../adr/ADR-001-simulador-sin-calculo-propio.md).

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-09-16 | Iker Acevedo | Documentación inicial de los modelos. |
| 2026-09-16 | Iker Acevedo | Corrección del catálogo de `MotivoNoLiquidacion`: la versión anterior tenía motivos inventados (`sobrepeso_sin_regla`, `error_api`) en snake_case que el backend nunca produce — un documento real de auditoría nunca hubiera matcheado ningún caso del mapeo. Ahora son los 9 valores reales, en el mismo PascalCase que el backend. |
