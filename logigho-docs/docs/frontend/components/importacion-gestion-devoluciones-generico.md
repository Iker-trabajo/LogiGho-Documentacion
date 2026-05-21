---

## Autor: Adalberto Gonzalez

Fecha creacion: 2026-05-16  
Estado: desarrollo  
Tipo: componente

# Componente: ImportacionDevolucionesGenerico

**Selector:** `app-importacion-devoluciones-generico`  
**Ubicación:** `src/app/components/importacion-devoluciones-generico`  
**Acceso:** Autenticado — se renderiza dentro de `GestionDevolucionesComponent`

---

## ¿Qué hace?

Modal reutilizable para importar archivos Excel de devoluciones. Recibe la estrategia de la transportadora activa y se adapta completamente a ella: valida columnas, mapea headers, transforma valores y envía los datos al backend en lotes de 5 000 registros. Por cada lote dispara primero la lambda `cargaDevoluciones` para que el backend ejecute procesado adicional, y luego realiza el insert en la colección vía método genérico.

---

## Decoradores

| Decorador  | Nombre              | Tipo                   | Descripción                                                           |
| ---------- | ------------------- | ---------------------- | --------------------------------------------------------------------- |
| `@Input`   | `strategy`          | `DevolucionStrategy`   | Estrategia activa. Define columnas y colección destino.               |
| `@Input`   | `isOpen`            | `boolean`              | Controla si el modal está visible.                                    |
| `@Input`   | `columnasRequeridas`| `string[]`             | Headers del Excel que se muestran como chips en la UI del modal.      |
| `@Output`  | `closeEvent`        | `EventEmitter<void>`   | Emitido al cerrar el modal (botón cerrar o tras carga exitosa).       |
| `@Output`  | `refreshDataEvent`  | `EventEmitter<void>`   | Emitido tras inserción exitosa para que el padre recargue la tabla.   |

---

## Propiedades clave

| Propiedad        | Tipo      | Descripción                                                             |
| ---------------- | --------- | ----------------------------------------------------------------------- |
| `uploadProgress` | `number`  | Porcentaje de progreso (0–100) mostrado en la barra de la UI.           |
| `currentBatch`   | `number`  | Lote en proceso actualmente, para el label "Lote X de Y".              |
| `totalBatches`   | `number`  | Total de lotes calculados al inicio de la inserción.                   |

---

## Servicios y endpoints

| Servicio                 | Método            | Endpoint                                              | Cuándo                          |
| ------------------------ | ----------------- | ----------------------------------------------------- | ------------------------------- |
| `ConsumoGenericoService` | `insertarGenerico`| `POST gestiondevoluciones`                            | Por cada lote (procesado backend — error no bloquea) |
| `ConsumoGenericoService` | `insertarGenerico`| `POST metodoGenerico?coleccion=<coleccion>`           | Por cada lote (persistencia en MongoDB) |

---

## Flujo principal

```
subirArchivo()
  -> FileReader.readAsBinaryString()
  -> XLSX.read() -> sheet_to_json(header: 1)
  -> validar columnas contra strategy.columnMapping
  -> mapear headers a camelCase
  -> strategy.transformarValor() por cada celda
  -> insertarPorLotes(jsonData)  [lotes de 5 000]
       -> POST gestiondevoluciones          (lambda de procesado — error se loguea, no bloquea)
       -> POST metodoGenerico?coleccion=<coleccion>  (insert en MongoDB)
       -> avanzar uploadProgress por lote
  -> refreshDataEvent.emit()
  -> closeEvent.emit()
```

---

## Historial de cambios

| Fecha      | Autor              | Cambio                                                                        |
| ---------- | ------------------ | ----------------------------------------------------------------------------- |
| 2026-05-16 | Adalberto Gonzalez | Creación del componente genérico de importación                               |
| 2026-05-16 | Adalberto Gonzalez | Lambda `cargaDevoluciones` fija para todos los casos, previa al insert      |
| 2026-05-20 | Adalberto Gonzalez | Se aplico un delay de 1000ms antes de hacer el llamado a la Lambda `cargaDevoluciones` para tener tiempo de tener cargados los datos en la BD   |

---

## Observaciones

- Si el archivo tiene columnas extra (no mapeadas en `columnMapping` ni en `columnasOpcionales`), la validación falla — esto es intencional para evitar insertar campos inesperados.
- La colección destino es siempre `strategy.coleccion`, que varía por transportadora (ej: `GestionDevolucionesInterrapidisimo`).
