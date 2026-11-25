## ADR-001: la exportación a S3 lee por rango de `_id`, no con `Skip`

Fecha: 2026-11-24 · Estado: aceptada (pendiente de despliegue) · Spec 037 · Dominio core

## Contexto

`ApiLambdaGuardaTablaS3AOT` exporta colecciones completas a CSV. Desde el 2026-11-21 calcula primero los encabezados de toda la colección y luego lee las filas por páginas de 50.000 con `Sort(_id desc) + Skip + Limit` y un `CountDocuments` en cada página. Con `LiquidacionesLogigho` (1.725.552 documentos, 35 páginas) y `PedidosInter` (973.267 documentos, 20 páginas) la Lambda llegó a los 900 s y el Step `Step_Procesos_Logigho` fallaba.

Medido en producción, solo lectura, 2026-11-24:

| Posición de la página | Tiempo de leer 50.000 documentos con `Skip` |
| --------------------- | ------------------------------------------- |
| inicio (skip 0) | 3,8 s |
| mitad (skip 850.000) | 11,9 s |
| final (skip 1.650.000) | 21,0 s |

Todos los `_id` de ambas colecciones son `ObjectId` (verificado en producción y PreProd).

## Decisión

1. Leer las filas por rango: cada página pide `_id` menor que el último de la anterior, con el mismo orden descendente. El costo de una página deja de depender de su posición.
2. No contar la colección en cada página; el conteo estimado (metadatos) se usa una sola vez, al final, como control de integridad.
3. Pedir la página siguiente a Mongo mientras se convierte y sube la actual.
4. **No cambiar el formato del CSV**: mismos encabezados globales, mismo orden, mismas reglas de aplanado. Se verificó con pruebas que el archivo es idéntico byte a byte al del algoritmo anterior.
5. Crear un método nuevo en el repositorio y dejar intacto `ObtenerDocumentoParaleloS3Async`, que usan otras Lambdas.

## Alternativas descartadas

- **Subir timeout o memoria:** el timeout ya está en el máximo de Lambda (900 s) y la memoria usó menos de 1 GB de 10 GB.
- **Un solo cursor abierto de principio a fin:** menos reintentos posibles ante cortes de red; el rango por `_id` se puede reanudar.
- **Encabezado calculado por página (diseño previo al PR 749 del 2026-11-21):** desalinea las columnas si una página trae campos distintos; se mantiene el encabezado global que ya corre en producción.
- **Dividir el trabajo en varias invocaciones:** el contrato de entrada no admite rangos; cambiarlo afecta al Step.

## Consecuencias

- Tiempo esperado (estimación, aún sin medir dentro de la Lambda en producción): unos 7 s por página, es decir 4-5 minutos más la pasada de encabezados para `LiquidacionesLogigho`.
- La pasada de encabezados sigue leyendo toda la colección: es el siguiente cuello de botella (mejora M-402).
- Si algún día un `_id` no es `ObjectId`, la exportación falla con un mensaje claro en vez de omitir documentos.
- Los documentos que se inserten mientras corre la exportación quedan fuera de esa corrida (antes desplazaban las páginas y podían duplicar o saltar filas).
