## Autor: Iker Acevedo

Fecha creacion: 2026-10-18  
Estado: desarrollo

---

# Caso de uso: RecibirResultadoUseCase

`Aplicacion/CasosUso/RecibirResultadoUseCase.cs` — ruta `/webhooks/ws-inter/flujo-guia`.

Procesa lo que ws-inter devuelve por cada guía solicitada.

## Reglas

- **Idempotente.** El mismo callback dos veces, o uno más viejo que el ya guardado (`consultado_en`), se ignora.
- **Correlación.** Primero por la `ref` que enviamos; si el proveedor no la devuelve, por número de guía. Si la `ref`
  pertenece a *otra* guía, el resultado se cuenta como inválido y no se mezcla.
- **Motivo nuevo solo si cambió.** Se compara el hash (`HuellaMotivos`) de la lista completa de motivos. Si cambió:
  `$set PedidosInter.MotivoNovedadInter` y nueva entrada en el historial (tope 50). Si no cambió, solo contadores y fechas.
- **Orden de escritura.** Primero `PedidosInter`, después `SeguimientoNovedadesInter`: si algo falla entre los dos, el
  reintento del proveedor vuelve a ver "hubo cambio" y completa lo que faltó.
- **Errores.** Cuerpo ilegible → `PayloadInvalidoException` (400). Resultado inválido o guía desconocida → se cuenta y
  la llamada sigue en 200 (reintentarlo no lo arreglaría). Falla de Mongo → excepción → 500 para que el proveedor reintente.

## Qué se extrae de la respuesta — `ExtractorMotivoNovedad`

- **Último motivo**: el de fecha más reciente de `motivos`.
- **Geo**: la del evento del `flujo` que trae ese motivo; si no tiene, la del último evento con geo.
- **Texto** de `MotivoNovedadInter`: `MOTIVO - URL de Google Maps` (solo `MOTIVO` si no hay geo). Si la geo no trae URL se
  construye `https://www.google.com/maps?q=lat,lng` con cultura invariante (punto decimal).
- Guía `encontrado:false` → solicitud `NoEncontrada`; no se toca el pedido.

## Pruebas

`test/ApiLambdaNovedadesInter.Tests/Aplicacion/RecibirResultadoUseCaseTests.cs` y `Dominio/ExtractorMotivoNovedadTests.cs`
(usan la respuesta real de la guía `240063616541` como fixture).
