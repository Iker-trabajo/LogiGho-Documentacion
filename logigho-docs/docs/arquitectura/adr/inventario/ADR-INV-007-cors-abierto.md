---

## Autor: Iker Acevedo  
Fecha creacion: 2026-10-19  
Estado: aceptada

# ADR-INV-007 — CORS abierto (`*`) en las rutas de inventario

**Autor:** Iker Acevedo  
**Fecha:** 2026-10-19  
**Estado:** Aceptada, con riesgo aceptado

---

## Contexto

La construcción inicial (INV-032) usaba una lista blanca `CORS_ORIGENES`, que obliga a declarar los dominios reales del front antes de repuntar; si faltaba alguno, el navegador descartaba respuestas de movimientos que sí se habían guardado. Las lambdas gemelas devolvían `*`.

## Opciones consideradas

### Opción A — Lista blanca de orígenes

**Pros:** superficie menor.  
**Contras:** riesgo operativo en la ventana si el dominio no coincide; trabajo extra por entorno.

### Opción B — `*` sin credenciales

**Pros:** igual que hoy, sin sorpresas en el front.  
**Contras:** cualquier origen puede leer respuestas si tiene un token.

---

## Decisión

**Se eligió:** Opción B, decisión de Iker del 2026-10-19, riesgo aceptado.

**Razón:** la identidad viaja en la cabecera `Token`, no en cookies; con `*` la lambda **no** emite `Access-Control-Allow-Credentials`. Está en Terraform de producción (`cors_origenes = ["*"]`) y en los recursos REST, que llevan `OPTIONS` por recurso. El E2E validó el preflight (OPTIONS 204, POST 200, sin errores de CORS).

---

## Consecuencias

**Positivas:** sin dependencia de dominios en el go-live.  
**Negativas:** el README de la lambda quedó desactualizado (dice que `*` se ignora). Volver a una lista blanca exige declarar los dominios del front y probar el preflight ([Operación — discrepancias](../../../guias/inventario-operacion.md#discrepancias-conocidas-entre-documentos-y-codigo)).

---

## Impacto en el código

| Módulo / Repo | Cambio |
|---|---|
| `ApiLambdaInventario` (`RespuestaHttp.cs`) | Acepta `CORS_ORIGENES="*"` y omite credenciales |
| Terraform `environments/prod/inventario` | `cors_origenes = ["*"]` |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-10-21 | Iker Acevedo | Documentado en MkDocs |

---

## Referencias

- [API de inventario — CORS](../../../api/inventario.md#cors)
