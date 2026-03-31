---
autor: Iker
fecha_creacion: 2026-03-31
ultima_actualizacion: 2026-03-31
estado: desarrollo
nivel: 2
---

# Worker: Parseo de Excel — Carga Masiva de Guías

**Archivo:** `carga-guias.worker.ts`
**Ubicación:** `SitioLogiGho/src/app/views/tienda/pedidos-masivo/workers`
**Consumidor:** `CargaGuiasComponent`

---

## ¿Qué hace?

Ejecuta en un hilo secundario del navegador (Web Worker) el parseo, validación y agrupación del archivo Excel de carga masiva. Al correr fuera del hilo principal, el usuario puede seguir interactuando con la UI mientras el archivo se procesa, sin que la página se congele.

---

## ¿Por qué un Web Worker?

El procesamiento de archivos Excel grandes en el hilo principal de JavaScript bloquea el event loop: la UI se congela, los clicks no responden y el navegador puede mostrar el mensaje "La página no responde". Un Web Worker corre en un hilo separado del sistema operativo, paralelo al hilo principal, con su propia memoria y sin acceso al DOM.

| Situación | Sin Worker | Con Worker |
|---|---|---|
| Parseo de 5000 filas con SheetJS | UI bloqueada varios segundos | UI completamente fluida |
| Deduplicación de registros | Congela animaciones y clicks | Invisible para el usuario |
| Archivo malformado | Puede bloquear la UI si hay un loop | El error se captura en el worker y se reporta limpiamente |
| Múltiples columnas (hasta 100) | Mayor riesgo de freeze | Sin impacto en UI |

**Transferencia de memoria sin copia:** El `ArrayBuffer` del archivo se transfiere al worker con `transfer` (no se copia). Esto significa que el hilo principal cede la propiedad del buffer al worker: más rápido y sin duplicar el uso de memoria RAM. Una vez transferido, el componente no puede acceder al buffer hasta que el worker termine.

---

## Estructura de archivos

```
workers/
└── carga-guias.worker.ts    Único worker del módulo pedidos-masivo
```

> Este worker es compartido potencialmente por componentes del módulo, aunque actualmente solo lo invoca `CargaGuiasComponent`.

---

## Contrato de comunicación

El worker se comunica con el componente vía `postMessage`. No tiene estado: cada mensaje es independiente.

### Mensaje de entrada (componente → worker)

```typescript
{
  buffer: ArrayBuffer,       // Contenido binario del archivo .xlsx (transferido, no copiado)
  fieldsToExclude: string[]  // Columnas a ignorar en la clave de deduplicación (FIELDS_TO_EXCLUDE del componente)
}
```

### Mensaje de salida — éxito (worker → componente)

```typescript
{
  ok: true,
  pedidosPorTienda: [string, any[]][],  // Array de entradas del Map<tiendaId, pedidos[]>
  totalPedidos: number                  // Total de pedidos únicos procesados
}
```

> El `Map` no es serializable por `postMessage`, por eso se envía como `[...grupos.entries()]` (array de pares `[clave, valor]`). El componente lo reconstruye como `Map`.

### Mensaje de salida — error (worker → componente)

```typescript
{
  ok: false,
  error: string   // Descripción legible del error
}
```

---

## Flujo de procesamiento

```
Recibe mensaje { buffer, fieldsToExclude }
  │
  ├─ 1. XLSX.read(buffer, { type: 'array' })
  │       → Parsea el workbook completo
  │       → Toma solo la primera hoja (SheetNames[0])
  │       → sheet_to_json con { header: 1 } → array 2D: any[][]
  │
  ├─ 2. Validaciones estructurales (fallo temprano, sin procesar filas)
  │       ├─ Archivo vacío: rawData.length < 2
  │       ├─ Supera MAX_ROWS (5000 filas)
  │       ├─ Sin encabezados válidos
  │       ├─ Supera MAX_COLUMNS (100 columnas)
  │       ├─ Faltan columnas requeridas: 'ID TIENDA', 'TELEFONO'
  │       └─ Columnas duplicadas
  │
  ├─ 3. Conversión a JSON + deduplicación
  │       → Itera filas desde índice 1 (salta encabezados)
  │       → Omite filas completamente vacías
  │       → Por cada celda string: trim() + elimina comas al final (replace(/,+$/, ''))
  │       → Construye clave de deduplicación:
  │             headers (sin fieldsToExclude) → valores de la fila → join('|')
  │       → Si la clave ya existe en seenKeys → fila descartada (duplicado)
  │       → Si es nueva → se agrega a jsonData y se registra en seenKeys
  │
  ├─ 4. Agrupación por tienda
  │       → Itera jsonData
  │       → Clave = pedido['ID TIENDA'].toString()
  │       → Si no tiene ID TIENDA → pedido descartado silenciosamente
  │       → grupos: Map<tiendaId, pedidos[]>
  │
  └─ 5. postMessage({ ok: true, pedidosPorTienda: [...grupos.entries()], totalPedidos })
```

---

## Validaciones del worker

| # | Validación | Condición de fallo | Mensaje de error |
|---|---|---|---|
| 1 | Archivo vacío | `rawData.length < 2` | `'El archivo esta vacio o solo tiene los encabezados'` |
| 2 | Límite de filas | `totalFilas > 5000` | `'El archivo supera 5000 filas, el archivo contiene N'` |
| 3 | Sin encabezados | `encabezadosValidos.length === 0` | `'El archivo no contiene encabezados validos'` |
| 4 | Límite de columnas | `encabezadosValidos.length > 100` | `'El archivo tiene mas columnas de las permitidas'` |
| 5 | Columnas requeridas | Falta `ID TIENDA` o `TELEFONO` | `'Faltan columnas requeridas: COLUMNA1, COLUMNA2'` |
| 6 | Columnas duplicadas | Mismo nombre de cabecera más de una vez | `'Columnas duplicadas COLUMNA1, COLUMNA2'` |

Todas las validaciones se evalúan antes de procesar las filas. Si alguna falla, el worker llama a `postMessage({ ok: false, error })` y retorna inmediatamente, sin continuar.

---

## Constantes internas

| Constante | Valor | Descripción |
|---|---|---|
| `MAX_ROWS` | `5000` | Límite máximo de filas de datos (excluyendo encabezados) |
| `MAX_COLUMNS` | `100` | Límite máximo de columnas en el archivo |
| `REQUIRED_COLUMNS` | `['ID TIENDA', 'TELEFONO']` | Columnas cuya presencia es obligatoria. Comparación en mayúsculas con trim |

---

## Lógica de deduplicación

La clave de deduplicación se construye concatenando con `|` los valores de **todas las columnas excepto las de `fieldsToExclude`** (los campos de estado/financiero que el componente inyecta al worker).

```
key = headers
  .filter(h => !fieldsToExclude.includes(h))
  .map(h => obj[h])
  .join('|')
```

**Implicación:** Dos filas son consideradas duplicadas si todos sus campos de negocio son idénticos, independientemente de los campos de estado o fecha. Un pedido copiado en el Excel con distintos valores de estado seguirá siendo detectado como duplicado.

---

## Dependencias

| Librería | Uso |
|---|---|
| `xlsx` (SheetJS) | Parseo del archivo `.xlsx` — `XLSX.read()` y `XLSX.utils.sheet_to_json()` |

> SheetJS no se importa en el hilo principal para este flujo: vive exclusivamente en el worker. Esto evita que su peso de bundle impacte el tiempo de carga inicial de la aplicación.

---

## Manejo de errores

El worker envuelve todo el procesamiento en un bloque `try/catch`. Cualquier excepción no contemplada (archivo corrupto, error interno de SheetJS, etc.) resulta en:

```typescript
postMessage({ ok: false, error: error?.message || 'Error al procesar el archivo' })
```

El componente `CargaGuiasComponent` verifica `response.ok` y muestra el mensaje en un Swal al usuario.

---

## Comportamientos silenciosos (sin error, sin aviso)

| Comportamiento | Descripción |
|---|---|
| Filas vacías omitidas | Una fila donde todas las celdas son `null`, `undefined` o `''` se descarta sin avisar |
| Filas duplicadas omitidas | Un pedido idéntico (según la clave de deduplicación) se descarta sin avisar |
| Pedidos sin `ID TIENDA` omitidos | Si un pedido no tiene valor en la columna `ID TIENDA`, se descarta al agrupar sin aviso |
| Comas al final de strings eliminadas | Los valores de texto con comas al final (`valor,`) son limpiados con `replace(/,+$/, '')` |

---

## Changelog

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-03-31 | Iker | Mejora en rendimiento: parseo del Excel movido a Web Worker para no bloquear UI |

---

## Observaciones

- **No tiene estado:** El worker procesa un mensaje y responde. No guarda nada entre llamadas.
- **Un solo worker por instancia:** `CargaGuiasComponent` instancia el worker con `new Worker(...)` y lo termina con `worker.terminate()` al finalizar. No hay pool de workers.
- **Map no serializable:** `postMessage` usa el algoritmo structured clone, que no soporta `Map`. Por eso el worker serializa el mapa como `[...grupos.entries()]` y el componente lo reconstruye.
- **Columnas requeridas case-insensitive:** La validación de `REQUIRED_COLUMNS` normaliza los encabezados a mayúsculas con `.toUpperCase()` antes de comparar, por lo que `id tienda` o `Id Tienda` son aceptados.
- **Primera hoja solamente:** El worker solo procesa `wb.SheetNames[0]`. Si el archivo tiene múltiples hojas, las adicionales se ignoran sin aviso.
- **`fieldsToExclude` viene del componente:** El worker no tiene la lista hardcodeada; la recibe como parámetro. Esto permite que el componente controle qué columnas se consideran para deduplicación sin modificar el worker.
