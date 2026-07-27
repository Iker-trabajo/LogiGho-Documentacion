## Autor: Iker Acevedo

Fecha creacion: 2026-07-13

Estado: produccion

# Integración Pancake — Visión general

## ¿Qué es?

Es un proceso automatizado que corre 4 veces al día: recolecta las estadísticas de campañas publicitarias de **todas las páginas conectadas en Pancake** y las almacena en **MongoDB** y en la **RDS MySQL**.

Todo el proceso está **orquestado con AWS Step Functions** y **agendado con EventBridge Scheduler** — no requiere intervención manual: corre solo a las 7:00 am, 9:00 am, 2:00 pm y 5:00 pm (hora Colombia).

---

## Objetivo de negocio

Tener historificadas, por **franja del día** y por **campaña**, las métricas publicitarias de cada página (gasto, clics, CPM, CTR, alcance, resultados, etc.) para poder analizarlas y cruzarlas con el resto de la operación.

- **Un "cierre" diario** (7:00 am) consolida los números del **día anterior**.
- **Tres fotos intradía** (9:00 am, 2:00 pm, 5:00 pm) capturan cómo evoluciona el **día en curso**.

---

## Arquitectura de un vistazo

```
                     EventBridge Scheduler  (America/Bogota)
                     7:00am · 9:00am · 2:00pm · 5:00pm
                                   │
                                   ▼  { slot_id, tipo_calculo }
          ┌───────────────────────────────────────────────────┐
          │   Step Function:  PipelineEstadisticasPancake     │
          └───────────────────────────────────────────────────┘
                                   │
  1) ApiLambdaCalcularVentanaTiempo ──► $.ventana { since, until, fecha_reporte, tipo, slot_id }
                                   │
  2) ApiLambdaObtenerCuentasPrincipales ──► $.cuentas [ { cuenta_id, token_acceso } ]
                                   │
  3) Map  (por cuenta madre · MaxConcurrency 3)
        └── ApiLambdaListarPaginasPancake ──► MongoDB: PancakePaginas
                                   │
  4) ApiLambdaObtenerPaginasActivas ──► $.paginasActivas [ { page_id, page_access_token, timezone } ]
                                   │
  5) Map  (por página · MaxConcurrency 5 · ItemSelector inyecta la ventana)
        └── ApiLambdaObtenerEstadisticas
                 ├── MongoDB:          PancakeEstadisticasPaginas
                 └── RDS Aurora MySQL: PancakeEstadisticasPaginas   (doble escritura)
```

> El detalle de la orquestación (definición ASL, los 2 `Map`, `ItemSelector`, `Retry`/`Catch`) está en **[Orquestación](orquestacion.md)**.

---

## Las 5 lambdas de un vistazo


| #   | Lambda                                                                              | Rol en el pipeline                                                                                                                   | Lee / Escribe                                        |
| --- | ----------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------- |
| 1   | **[ApiLambdaCalcularVentanaTiempo](lambdas/01-calcular-ventana-tiempo.md)**         | Calcula la ventana de tiempo (`since`/`until`) según el tipo de corrida — los parámetros necesarios para consumir la API de Pancake. | (cálculo puro)                                       |
| 2   | **[ApiLambdaObtenerCuentasPrincipales](lambdas/02-obtener-cuentas-principales.md)** | Devuelve las cuentas madre activas con su token                                                                                      | Lee `PancakeCuentasPrincipales`                      |
| 3   | **[ApiLambdaListarPaginasPancake](lambdas/03-listar-paginas-pancake.md)**           | Trae y sincroniza las páginas de cada cuenta madre                                                                                   | Escribe `PancakePaginas` (Mongo + MySQL)             |
| 4   | **[ApiLambdaObtenerPaginasActivas](lambdas/04-obtener-paginas-activas.md)**         | Devuelve las páginas activas con token para el 2º Map                                                                                | Lee `PancakePaginas`                                 |
| 5   | **[ApiLambdaObtenerEstadisticas](lambdas/05-obtener-estadisticas.md)**              | Trae las estadísticas por campaña y las guarda (doble escritura)                                                                     | Escribe `PancakeEstadisticasPaginas` (Mongo + MySQL) |


> **Fuera del pipeline** hay además un **endpoint on-demand**: **[ApiLambdaConsultarEstadisticasPagina](endpoint-consultar-estadisticas.md)** — un `POST` de API Gateway que trae estadísticas **frescas** de UNA página (hoy / ayer / rango personalizado), **sin persistir**. Para cuando alguien duda de una tienda y quiere ver los números al instante.

---

## Tecnologías


| Área                    | Tecnología                                                              |
| ----------------------- | ----------------------------------------------------------------------- |
| Lenguaje / arquitectura | **.NET 8**, Clean Architecture (Dominio / Aplicación / Infraestructura) |
| Cómputo                 | AWS Lambda (runtime `dotnet8`, `Zip`, `x86_64`)                         |
| Orquestación            | AWS Step Functions (tipo **Standard**)                                  |
| Agendamiento            | Amazon EventBridge Scheduler (con zona horaria `America/Bogota`)        |
| Persistencia            | MongoDB + **Aurora MySQL** (driver `MySqlConnector`)                    |
| Seguridad               | Cadenas de conexión cifradas **AES-256-ECB** (módulo `Seguridad`)       |
| API externa             | Pancake                                                                 |


---

## Colecciones y tablas


| Nombre                       | Motor               | Rol                                                         |
| ---------------------------- | ------------------- | ----------------------------------------------------------- |
| `PancakeCuentasPrincipales`  | MongoDB             | **Entrada**: cuentas madre con su `tokenAcceso` y `estado`  |
| `PancakePaginas`             | MongoDB **y** MySQL | Páginas sincronizadas (activas e inactivas), clave `pageId` |
| `PancakeEstadisticasPaginas` | MongoDB **y** MySQL | **Salida**: un registro por campaña, por franja del día     |


---

## Convenciones clave

- **Nombres:** "cuenta madre" y "cuenta principal" son el mismo concepto — la colección se llama `PancakeCuentasPrincipales`, pero en el código y en esta documentación se le suele decir "cuenta madre" por ser la que agrupa varias páginas.
- **Casing:** los datos de MongoDB viajan en `camelCase`; el pipeline y AWS usan `snake_case` (`[JsonPropertyName]`). Cada DTO traduce entre ambos mundos para evitar errores de ortografía.
- **Fechas:** la matemática de días se hace en hora **Colombia** (`America/Bogota`, UTC-5) y se **almacena en UTC** (formato Unix).
- **Seguridad:** los tokens de Pancake viajan en la query string → **nunca se loguea la URL completa**, solo el `page_id`.
- **Idempotencia:** las escrituras usan **UPSERT** por clave de negocio, así una corrida repetida actualiza en lugar de duplicar.

---

## Historial de cambios


| Fecha      | Autor        | Cambio                                                                                                                                                                   |
| ---------- | ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 2026-07-13 | Iker Acevedo | Construcción e implementación completa de la integración: 5 lambdas, Step Function, EventBridge y doble escritura Mongo + Aurora MySQL.                                  |
| 2026-07-27 | Iker Acevedo | `ListarPaginasPancake` con doble escritura Mongo + MySQL; fix ventana de cierre (`until` a `23:59:59`); nuevo endpoint on-demand `ApiLambdaConsultarEstadisticasPagina`. |


