## Autor: Iker Acevedo
Fecha creación: 2026-09-16

Estado: preprod

# Operación — correr una liquidación de prueba en PreProd

---

## Liquidar todas las guías pendientes

**Consola AWS → Step Functions → `Financiero-PreProd` → Start execution.**

En el cuadro **Input**:

```json
{}
```

---

## Liquidar una guía puntual (o varias)

Útil para probar un caso específico sin esperar a que se acumulen pendientes, o para repetir una guía después de corregir su configuración.

```json
{
  "Guias": ["700000001", "700000002"]
}
```

---

## Mezclar transportadoras legacy y TCC en la misma corrida

No hace falta separar la ejecución por transportadora — `Particionar()` separa internamente cada pedido según su `ConfiguracionLiquidacionTransportadora`. Un mismo `{}` (o una misma lista de guías mixta) liquida en paralelo lo que le toca a Legacy (Inter/Envía/Servientrega) y lo que le toca al motor Dinámico (TCC), y todo termina en la misma colección `LiquidacionesLogigho`.

---

## Verificar resultado

| Qué revisar | Dónde | Comando/Nota |
|---|---|---|
| Estado ejecución | AWS console: Step Functions → `Financiero-PreProd` → **Executions** | Verde = `Succeeded` |
| Logs Lambda | CloudWatch → `/aws/lambda/ApiLambdaLiquidacionesLogighoAOT` | Busca errores, tiempos |
| Documentos creados | MongoDB preprod → `LiquidacionesLogigho` | `db.LiquidacionesLogigho.find({Numeropreenvio: "700000001"})` |
| Guías no liquidadas (Motor Dinámico) | Frontend: [Auditoría Excepciones](../../../../frontend/views/director-de-operaciones/costos-transportadora/components/auditoria-excepciones.md) | Hoy vacía (escrita en fases futuras) |
| Resumen cuenta Mongo | MongoDB compass/mongosh | `db.LiquidacionesLogigho.countDocuments()` |

---

## Troubleshooting

| Síntoma | Causa | Solución |
|---|---|---|
| `JsonSerializationException: Cannot create and populate list type ReadOnlyCollection...` | Modelo configuración usa `IReadOnlyList<T>` (Newtonsoft no lo deserializa directo) | Cambiar a `List<T>`. Ya corregido en commit `892e372`. |
| `An invalid request URI was provided... BaseAddress must be set` | Falta `URL_SERVICIO_AWS` + pedido no-TCC en misma ejecución | Verificar variable en template CloudFormation, redesplegar. |
| Merge markers `<<<<<<<`/`=======`/`>>>>>>>` en `.yaml` | Conflicto `git stash pop` no resuelto (no error sintaxis) | `grep -rln "<<<<<<<" .preprod-hub`, resolver, redesplegar. |
| `Invalid template path` en `aws cloudformation deploy` | Ruta relativa desde directorio equivocado | Ejecutar desde raíz repo o usar ruta absoluta al `.yaml`. |
| Liquidación termina pero 0 documentos en Mongo | Pedido sin tienda, sin números guía, o tipos indeterminados | Revisar logs CloudWatch por `MotivoNoLiquidacion`. Ver [Validación](motor-dinamico/validacion.md). |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-09-16 | Iker Acevedo | Guía inicial de operación: ejecución manual, verificación y troubleshooting de los errores reales vistos durante la puesta en marcha. |
