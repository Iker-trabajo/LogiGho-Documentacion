## Autor: Iker Acevedo
Fecha creacion: 2026-07-13

Estado: produccion

## Lambda: ApiLambdaObtenerPaginasActivas

**Accionador:** Step Functions (`Task`)

**AOT:** No

**Posición en el pipeline:** 4 de 5

---

## ¿Qué hace?

Lee la colección `PancakePaginas` (que la lambda 3 acaba de sincronizar), se queda con las páginas **activadas que tienen token**, y devuelve un array liviano `{ page_id, page_access_token, timezone }` que alimenta el **2º `Map`** (una iteración por página).

Es el "puente" entre las páginas guardadas y la recolección de estadísticas: descarta lo que no sirve (inactivas o sin token) para no gastar iteraciones en vano.

---

## Request

No recibe datos propios (su salida se guarda en `$.paginasActivas`).

```json
{}
```

---

## Response

Un **array** de páginas listas para consultar estadísticas:

```json
[
  {
    "page_id": "850833678102371",
    "page_access_token": "eyJhbGciOiJ...",
    "timezone": -5
  }
]
```

| Campo | Tipo | Descripción |
| ----- | ---- | ----------- |
| `page_id` | `string` | Id de la página en Pancake |
| `page_access_token` | `string` | Token de la página (para consultar sus estadísticas) |
| `timezone` | `number` | Zona horaria de la página |

> En el Step Function esta salida se guarda en `$.paginasActivas` y alimenta el **2º `Map`** (MaxConcurrency 5).

---

## Flujo interno

```
FunctionHandler (Function.cs)
  -> ObtenerPaginasActivasUseCase.EjecutarAsync()
       -> IPaginasRepository.ObtenerActivasAsync()
            -> MongoDB: PancakePaginas
               Filter.Eq(activada, true)
               (descarta las que no tengan tokenAccesoPagina)
               Proyección -> { page_id, page_access_token, timezone }
  -> [RESULT] IReadOnlyList<PaginaActiva>
```

---

## Arquitectura Clean Architecture

```
ApiLambdaObtenerPaginasActivas/
├── Function.cs
├── Dominio/
│   └── Entidades/
│       └── PaginaActiva.cs              ← { page_id, page_access_token, timezone }
├── Aplicacion/
│   ├── CasosUso/
│   │   └── ObtenerPaginasActivasUseCase.cs
│   └── Interfaces/
│       └── Repositorios/ IPaginasRepository.cs
└── Infraestructura/
    └── Repositorio/ DocumentRepository.cs   ← MongoDB (lectura + desencriptación)
```

---

## Variables de entorno

| Variable | Descripción | Valor ejemplo |
| -------- | ----------- | ------------- |
| `CADENA_CONEXION` | Cadena de conexión MongoDB (cifrada AES-256-ECB) | String cifrado |
| `DATABASE_NAME` | Nombre de la base de datos MongoDB | `"LogighoDB"` |

---

## Configuración Lambda

| Parámetro | Valor |
| --------- | ----- |
| Runtime | `dotnet8` |
| Handler | `ApiLambdaObtenerPaginasActivas::ApiLambdaObtenerPaginasActivas.Function::FunctionHandler` |
| Memory | `256 MB` |
| Timeout | `30 segundos` |
| Architecture | `x86_64` |

---

## Historial de cambios

| Fecha | Autor | Cambio |
| ----- | ----- | ------ |
| 2026-07-13 | Iker Acevedo | Creación. Devuelve páginas activas con token para el 2º Map. `timezone` modelado como `double?` para tolerar valores sucios de Pancake. |

---

## Observaciones

- Se separó a propósito de la lambda 5: **listar** las páginas activas (una consulta) y **consultar estadísticas** (N llamadas concurrentes) son responsabilidades distintas. Mantenerlas separadas permite que el `Map` controle la concurrencia sobre una lista ya filtrada.
- Las páginas sin `tokenAccesoPagina` se descartan aquí — así no llegan al 2º Map a fallar.
