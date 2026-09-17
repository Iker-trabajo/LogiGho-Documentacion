## Autor: Iker Acevedo
Fecha creación: 2026-09-16
Estado: producción
Tipo: vista

# Liquidaciones (Power BI)

**Selector:** `app-liquidaciones`
**Ubicación:** `src/app/views/ceo/analytics/liquidaciones`
**Acceso:** Autenticado

---

## ¿Qué hace?

Embebe un reporte de Power BI con el detalle de liquidaciones — el consumo, no la configuración. Es la única pantalla del front que **lee** el resultado de los dos motores de liquidación (Legacy y Dinámico); no distingue de cuál de los dos vino cada documento, porque ambos escriben en la misma colección con el mismo formato (ver [Motor Dinámico → Infraestructura](../../../backend/lambdas-dotnet/lambdas/ApiLambdaLiquidacionesLogighoAOT/motor-dinamico/infraestructura.md)).

---

## Filtro por tiendas asignadas

Al entrar, lee `tiendas_asignadas` de `sessionStorage`. Si el usuario tiene la tienda especial `"Todas"` asignada, carga el reporte sin filtro; si no, arma un filtro básico de Power BI (`IBasicFilter`) sobre la tabla `00_Tienda_Logi` para que cada usuario solo vea las liquidaciones de sus propias tiendas.

El token de acceso a Power BI se obtiene vía `ConsumoGenericoService.insertarGenerico("1", "authAzure")` — el mismo patrón de autenticación Azure AD que usa el resto de los reportes embebidos de la plataforma.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-09-16 | Iker Acevedo | Documentación inicial. |
