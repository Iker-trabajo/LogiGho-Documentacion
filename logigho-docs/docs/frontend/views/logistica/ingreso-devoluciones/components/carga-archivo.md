## Autor: Iker Acevedo

Fecha creacion: 2026-08-26

Estado: produccion

## Componente: carga-archivo

**Selector:** `app-carga-archivo`
**Ubicación:** `components/carga-archivo/`

---

## ¿Qué hace?

Lee un archivo Excel/CSV con una sola columna de guías, lo depura, y le entrega al padre la lista final. **No llama al backend** — eso lo hace el orquestador (`ingreso-devoluciones.component.ts`) al recibir el `@Output`. Separar la lectura del archivo de la llamada HTTP hace este componente testeable y reusable sin mockear red (Single Responsibility).

```
@Output() loteListo: EventEmitter<string[]>
```

---

## Flujo

```
onDrop / onFileInputChange
  manejarArchivo(archivo)
    valida extension (.xlsx, .xls, .csv, .txt) — corta ahi si no matchea
    leerArchivo(archivo)
      FileReader.readAsBinaryString
      XLSX.read(...) + sheet_to_json(ws, { header: 1, raw: false })
      detecta si la primera fila es encabezado (esCandidataGuia sobre la primera celda)
      depurarGuias(crudo)              — de ingreso-devoluciones.rules.ts
      resultado: GuiasDepuradas { guias, vacias, duplicadas, excedeTope }

confirmar()
  si hay guias validas y no excede el tope -> loteListo.emit(resultado.guias)
```

⚠️ **Bug real corregido (2026-08-26):** `confirmar()` tenía la condición de guarda invertida — `if (!resultado || resultado.guias.length || excedeTope) return;` cortaba precisamente cuando **sí** había guías válidas (`length > 0` es verdadero), así que el botón "Procesar" nunca emitía nada útil. El botón del HTML sí tenía el `[disabled]` correcto (`guias.length === 0`), pero el método en sí bloqueaba el caso contrario. Corregido a `!resultado.guias.length`.

---

## `raw: false` en `sheet_to_json` — por qué

```typescript
XLSX.utils.sheet_to_json(ws, { header: 1, raw: false });
```

SheetJS entrega el valor **formateado como se ve en la celda**, no un `Number` de JS re-convertido a texto. Si el Excel original guardó la guía como número (no texto), Excel ya le comió los ceros a la izquierda **antes** de que el archivo llegue a este componente — eso no se puede recuperar en el front, el dato ya se perdió en el origen. `raw: false` solo evita empeorarlo (por ejemplo, notación científica en guías muy largas), no lo arregla.

---

## Descarga de plantilla

```typescript
descargarPlantilla()
  GetObjectService.obtenerObjeto('logigho-plantillas', 'InventarioDevolucion.xlsx', 'false', 'true')
  descargarBase64Xlsx(respuesta.resultado, 'InventarioDevolucion.xlsx')   arma un Blob local y dispara la descarga
```

⚠️ **Error corregido:** la primera versión usaba `getMultiObject` (vía `insertarGenerico`), que espera un **arreglo** de `objectName` — de ahí el "Multi". Con un solo nombre suelto, la llamada fallaba. Se corrigió a `GetObjectService.obtenerObjeto` (singular), el mismo servicio que usan `importacion-facturas` y `anular-guias` para sus plantillas.

Se pide en base64 y se arma un `Blob` local (en vez de abrir una URL firmada en otra pestaña) para que el archivo se descargue con su nombre correcto, en vez de abrirse crudo en el navegador — con un `.xlsx` eso solo confunde al operario.

---

## Observaciones

- El archivo es de **una sola columna** — no hay selector de columna como en un flujo multi-columna genérico. La detección de encabezado (`esCandidataGuia` sobre la primera celda) es la única heurística necesaria.
- `depurarGuias` no consulta nada contra el backend — la deduplicación contra guías ya procesadas de corridas anteriores (idempotencia) la hace el `Worker`, no el front.
