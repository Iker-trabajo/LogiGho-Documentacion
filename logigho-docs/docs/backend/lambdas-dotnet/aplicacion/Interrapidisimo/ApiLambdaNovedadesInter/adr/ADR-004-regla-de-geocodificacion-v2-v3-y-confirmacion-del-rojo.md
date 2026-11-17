---

## Autor: Iker Acevedo  
Fecha creacion: 2026-11-06  
Estado: aceptada

# ADR-004 — Regla de geocodificación v2/v3 y confirmación del rojo

**Autor:** Iker Acevedo  
**Fecha:** 2026-11-06  
**Estado:** Aceptada (Iker 2026-11-06; PreProd validada; producción pendiente)

---

## Contexto

Distancias necesita ubicar la dirección del pedido con Google Geocoding. Las direcciones de Inter traen texto extra (apartamento, barrio,
observaciones), así que Google suele marcar `partial_match`. La primera regla (confiable solo sin coincidencia parcial) se midió con
**277 guías reales copiadas a PreProd** y dejó calculada apenas el **5 %** (tarea Q1 de la spec 012). Además, un reclamo a Inter por «novedad falsa»
basado en una geocodificación errónea sería costoso.

## Opciones consideradas

### Opción A — Mantener la regla inicial

Solo `partial_match = false`. **Pros:** muy conservadora. **Contras:** 5 % calculado; la herramienta casi no sirve.

### Opción B — Aceptar cualquier tipo de ubicación

**Pros:** cobertura máxima. **Contras:** `GEOMETRIC_CENTER` y `APPROXIMATE` dan la distancia al centro del barrio o la ciudad; distancias sin sentido.

### Opción C — Consulta limpia, aceptar `ROOFTOP`/`RANGE_INTERPOLATED` aunque sea parcial si la localidad = ciudad del pedido, y confirmar el rojo

**Pros:** cobertura 45 %; el único caso que genera un reclamo (rojo) se verifica con una segunda geocodificación. **Contras:** una llamada extra a Google en los rojos; el
criterio de ciudad por contención es laxo («CALI» dentro de «CALIMA», mejora M-263).

## Decisión

**Se eligió:** Opción C.

**Razón (medida):** con la regla inicial se calculaba el **5 %**; con la nueva, el **45 %** sobre las mismas **277 guías** de PreProd (125 calculadas, 0 en `Error`).

Reglas (detalle en [Reglas de la distancia](../dominio/reglas-distancia.md)):

1. Consulta principal `"{dirección limpia}, {ciudad}, Colombia"`, sin departamento, cortada en la primera palabra de unidad u observación.
2. Confiable si `ROOFTOP` o `RANGE_INTERPOLATED` (aunque sea parcial) **y** la localidad coincide con la ciudad del pedido. `APPROXIMATE` y `GEOMETRIC_CENTER`: «Dirección por validar».
3. **Confirmación del rojo:** si la ruta da más de 500 m, una segunda geocodificación con la dirección completa debe ser confiable y caer a ≤ 150 m de la primera; si no, la guía no entra al reporte 1 (`BajaConfianza`). En PreProd se confirmaron 38 rojas.
4. **v3:** Google omite `distanceMeters` cuando la ruta mide 0 m; se interpreta como 0 m y no como «sin ruta».
5. La **versión de la regla** viaja en la huella de la dirección y en `HuellaDistancias` (`"{HuellaMotivos}|v3"`): subirla recalcula todo una vez.

## Consecuencias

**Positivas:** cobertura útil; los reclamos a Inter se apoyan en dos geocodificaciones que coinciden; recalcular tras un cambio de regla es automático y una sola vez.  
**Negativas:** costo extra en rojos; una ciudad contenida en otra pasaría el filtro (M-263); si Google falla en la confirmación, la guía espera a la siguiente corrida.

## Impacto en el código

| Módulo / Repo | Cambio |
| ------------- | ------ |
| `Lambdas_NET_LogiGho` | `ReglasDistancia.cs`, `ReglasDireccion.cs`, `CalcularDistanciasUseCase.cs` (PR 623 y 624) |
| `LogighoDB` | `DireccionPedido.Localidades` y `ConfirmacionDireccion` (opcionales) |

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-11-06 | Iker Acevedo | Creación (regla v2, corregida a v3 el mismo día) |

## Referencias

- [ADR-003](ADR-003-distancias-y-reportes-en-el-mismo-proyecto.md) · [Distancias](../ApiLambdaNovedadesInterDistancias.md)
- Decisiones 2026-11-06 del workspace (`docs/architecture/decisiones.md`); spec 012 §3.
