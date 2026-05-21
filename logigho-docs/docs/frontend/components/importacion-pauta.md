---

## Autor: Adalberto González

Fecha creacion: 2026-05-21  
Estado: desarrollo  
Tipo: componente

# Componente: ImportacionPautaComponent

**Selector:** `app-importacion-pauta`  
**Ubicación:** `src/app/components/importacion-pauta`  
**Acceso:** Autenticado | Rol: `tienda`

---

## ¿Qué hace?

Modal que permite al usuario importar registros de pauta publicitaria desde un archivo Excel (.xlsx / .xls) hacia la colección `Pauta` en MongoDB. Valida la estructura del archivo antes de insertar: verifica que existan las columnas obligatorias y que ninguna fila tenga datos críticos vacíos. Si detecta errores, aborta sin insertar nada y notifica al usuario.

---

## Decoradores

| Decorador | Nombre            | Tipo                 | Descripción                                              |
| --------- | ----------------- | -------------------- | -------------------------------------------------------- |
| `@Input`  | `isOpen`          | `boolean`            | Controla si el modal está visible                        |
| `@Output` | `closeEvent`      | `EventEmitter<void>` | Emite cuando el usuario cierra el modal                  |
| `@Output` | `refreshDataEvent`| `EventEmitter<void>` | Emite tras importación exitosa para recargar la tabla    |
| `@ViewChild` | `archivoInput` | `ElementRef`         | Referencia al input file para limpiarlo tras importar    |

---

## Propiedades clave

| Propiedad      | Tipo          | Descripción                                              |
| -------------- | ------------- | -------------------------------------------------------- |
| `selectedFile` | `File \| null` | Archivo seleccionado por el usuario (controla el botón) |
| `fileToUpload` | `File \| null` | Archivo que se procesará al confirmar                   |
| `isLoading`    | `boolean`     | Bloquea el botón Importar mientras se procesa           |

---

## Servicios y endpoints

| Servicio                  | Método               | Endpoint                            | Cuándo                        |
| ------------------------- | -------------------- | ----------------------------------- | ----------------------------- |
| `ConsumoGenericoService`  | `insertarGenerico()` | `POST metodoGenerico?coleccion=Pauta` | Al confirmar la importación |

---

## Flujo principal

```
onFileChange()
  -> captura File en selectedFile y fileToUpload
  -> habilita botón Importar

uploadFile()
  -> guarda contra doble clic (isLoading)
  -> FileReader.readAsBinaryString()
  -> XLSX.read() → sheet_to_json()
  -> valida que existan columnas: Fecha, Idtienda, Tienda, Meta
     -> si faltan → Swal error → abort
  -> map() de filas: normaliza Fecha (serial numérico o DD/MM/YYYY → YYYY-MM-DD)
  -> detecta filas con campos obligatorios vacíos
     -> si hay → Swal warning con conteo → abort sin insertar
  -> insertarGenerico() → POST a Pauta
  -> Swal éxito → close() → refreshDataEvent.emit()
```

---

## Historial de cambios

| Fecha      | Autor          | Cambio                                              |
| ---------- | -------------- | --------------------------------------------------- |
| 2026-05-21 | Adalberto González | Creación del componente con importación básica      |
| 2026-05-21 | Adalberto González | Validación de headers obligatorios y filas inválidas |
| 2026-05-21 | Adalberto González | Botón Importar deshabilitado sin archivo seleccionado |

---

## Observaciones

> Deuda técnica, comportamientos no obvios, decisiones de diseño que no se ven en el código.

- La normalización de fecha maneja dos formatos de entrada del Excel: serial numérico (tipo `number`) y string `DD/MM/YYYY`. Si el Excel viene en otro formato, la fecha quedará como string sin transformar.
- `rowsMemory` está declarado pero no se usa actualmente — candidato a eliminar en próximo refactor.
