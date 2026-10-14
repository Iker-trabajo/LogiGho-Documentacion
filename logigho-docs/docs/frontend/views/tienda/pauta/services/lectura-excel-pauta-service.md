---

## Autor: Yadilton Torres
Fecha creacion: 2026-10-14
Estado: desarrollo

# Servicio: LecturaExcelPautaService

**Ubicación:** `src/app/views/tienda/pauta/services/lectura-excel-pauta.service.ts`  
**Scope:** `providedIn: 'root'`

---

## ¿Qué hace?

Lee el Excel de pauta con `XLSX` y revisa que tenga el formato de la plantilla. Devuelve cada fila con sus valores como texto para la previsualización del modal [Importacion Pauta](../../../../components/importacion-pauta.md). No valida los valores de cada fila (eso lo hace [`ValidacionFilasPautaService`](validacion-filas-pauta-service.md)) ni hace llamadas HTTP.

---

## Métodos

### `leerArchivo(archivo: File): Promise<ResultadoLecturaExcel>`

Lee la primera hoja del Excel.

| Parámetro | Tipo | Descripción |
| --------- | ---- | ----------- |
| `archivo` | `File` | Excel elegido o soltado en el modal |

**Proceso:**
1. Revisa que haya filas debajo de los encabezados.
2. Quita los espacios al inicio y al final de los encabezados (para que `"Meta "` no se guarde como un campo distinto a `Meta`) y los revisa con `validarEncabezados()`.
3. Quita las filas vacías, guarda el número de fila real del Excel (`filaExcel`) y normaliza la `Fecha` a `YYYY-MM-DD`: acepta el serial numérico de Excel y el texto `DD/MM/YYYY`.
4. Guarda cada valor como texto, para poder editarlo en la tabla (al importar se convierte al tipo correcto).

**Retorna:** un resultado con dos formas:

```typescript
type ResultadoLecturaExcel =
  | { ok: true; filas: FilaLeidaExcel[] }               // filaExcel + datos
  | { ok: false; titulo: string; detalle: string[] };   // error que se muestra en el modal
```

| Error | Título | Detalle |
|---|---|---|
| Sin datos | "El archivo no contiene datos" | Revisa que tenga registros debajo de los encabezados |
| Encabezados mal | "El archivo no tiene el formato de la plantilla" | La lista de `validarEncabezados()` |
| Archivo dañado | "No se pudo leer el archivo" | Verifica que sea un Excel válido |

### `validarEncabezados(headers, hayColumnasSinNombre)` *(privado)*

Revisa que el Excel tenga **exactamente** las columnas de `COLUMNAS_PAUTA`:

- Columnas faltantes.
- Columnas que no pertenecen a la plantilla (se guardarían como campos extra).
- Columnas repetidas (la segunda reemplazaría a la primera).
- Columnas con datos pero sin encabezado (se guardarían sin nombre de campo). Las columnas vacías a la derecha, que quedan por formato, se ignoran.

**Retorna:** la lista de problemas más la línea "Las columnas deben ser exactamente: … Usa la plantilla de descarga.", o `null` si todo está bien.

---

## Endpoints que consume

Ninguno.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-10-14 | Yadilton Torres | Servicio inicial: lectura del Excel y revisión de encabezados, separadas del modal. 3 pruebas. |

---

## Observaciones

- Una fecha en otro formato (ej. `2026/10/14`) no se normaliza: queda como texto y la validación de filas la marca como `Fecha no válida`.
