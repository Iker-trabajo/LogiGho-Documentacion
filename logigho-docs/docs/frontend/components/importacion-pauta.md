---

## Autor: Adalberto González

Fecha creacion: 2026-05-21  
Ultima actualizacion: 2026-10-14  
Estado: desarrollo  
Tipo: componente

# Componente: ImportacionPautaComponent

**Selector:** `app-importacion-pauta`  
**Ubicación:** `src/app/components/importacion-pauta`  
**Acceso:** Autenticado | Rol: `tienda`

---

## ¿Qué hace?

Modal que permite al usuario importar registros de pauta publicitaria desde un archivo Excel (.xlsx / .xls) hacia la colección `Pauta` en MongoDB.

Antes de guardar muestra una **previsualización editable**: cada celda se valida mientras el usuario escribe, los errores se marcan en rojo con un mensaje corto y solo se deja importar cuando no queda ningún error y todas las tiendas están verificadas. Si el archivo no tiene el formato de la plantilla, el error se muestra **dentro del modal** (con un temblor) y no se inserta nada.

Lo abre la vista [Pauta](../views/tienda/pauta/pauta.md) con el botón **Importar**.

---

## Estructura de archivos

```
importacion-pauta/
├── importacion-pauta.component.ts
├── importacion-pauta.component.html
├── importacion-pauta.component.scss          → Solo importa los archivos de estilos/
├── importacion-pauta.component.spec.ts       → 11 pruebas
└── estilos/
    ├── _estructura-modal.scss                → Overlay, caja del modal, header, footer y temblor
    ├── _carga-archivo-y-botones.scss         → Zona de arrastrar y soltar, tarjeta del archivo, errores y botones
    ├── _tabla-previsualizacion.scss          → Tabla, celdas con error / corregidas, mensajes y Deshacer
    ├── _paginacion-y-animaciones.scss        → Paginación de la previsualización y animaciones
    └── _diseno-celular.scss                  → Modal como hoja desde abajo y filas como tarjetas
```

La lógica que no es de la vista vive en la carpeta del módulo de Pauta (`views/tienda/pauta/`):

| Pieza | Para qué la usa el modal |
|---|---|
| [Modelos](../views/tienda/pauta/models/pauta-models.md) | `FilaPrevia`, `RegistroPauta` y las listas de columnas |
| [Lectura de Excel](../views/tienda/pauta/services/lectura-excel-pauta-service.md) | Leer el archivo y revisar los encabezados |
| [Validación de filas](../views/tienda/pauta/services/validacion-filas-pauta-service.md) | Validar cada fila y verificar las tiendas |
| [Repositorio](../views/tienda/pauta/repository/pauta-repository.md) | Insertar los registros |
| [Aviso toast](../views/tienda/pauta/services/aviso-toast-service.md) | Aviso "Solo escribe el Idtienda" |
| [Directiva solo números](../views/tienda/pauta/directives/campo-solo-numeros-directive.md) | Celdas de Idtienda y montos |

---

## Decoradores

| Decorador | Nombre            | Tipo                 | Descripción                                              |
| --------- | ----------------- | -------------------- | -------------------------------------------------------- |
| `@Input`  | `isOpen`          | `boolean`            | Controla si el modal está visible                        |
| `@Output` | `closeEvent`      | `EventEmitter<void>` | Emite cuando el usuario cierra el modal                  |
| `@Output` | `refreshDataEvent`| `EventEmitter<void>` | Emite tras importación exitosa para recargar la tabla    |
| `@ViewChild` | `archivoInput` | `ElementRef`         | Referencia al input file para limpiarlo tras importar    |
| `@HostListener` | `document:keydown.escape` | — | Escape cierra el modal (ver `onEscape()`) |

`ValidacionFilasPautaService` va en `providers` del componente: vive mientras vive el modal y no comparte con otras pantallas las tiendas ya consultadas.

---

## Propiedades clave

| Propiedad      | Tipo          | Descripción                                              |
| -------------- | ------------- | -------------------------------------------------------- |
| `selectedFile` | `File \| null` | Archivo seleccionado; muestra la tarjeta del archivo con `Cambiar` |
| `fileToUpload` | `File \| null` | Archivo que se procesará al confirmar                   |
| `isLoading`    | `boolean`     | `true` mientras se lee el archivo o se importa          |
| `errorArchivo` | `{ titulo, detalle[] } \| null` | Error del archivo que se muestra dentro del modal |
| `temblor`      | `'a' \| 'b' \| null` | Clase del temblor; alterna para que la animación se repita en errores seguidos |
| `arrastrando`  | `boolean`     | `true` mientras se arrastra un archivo sobre el modal   |
| `filasPrevia`  | `FilaPrevia[]` | Filas leídas del Excel, con sus valores editables y errores |
| `filasFiltradas` | `FilaPrevia[]` | Filas que se ven según "Solo errores". No se recalcula al editar, para que una fila no desaparezca mientras se corrige |
| `soloErrores`  | `boolean`     | Filtro "Solo errores" |
| `paginaActual` / `tamPagina` | `number` | Paginación de la previsualización (10 filas por página) |
| `hayPrevia`    | `boolean`     | El archivo se leyó bien y se muestra la previsualización (sigue en `true` aunque se eliminen todas las filas) |
| `hayCambios`   | `boolean`     | El usuario editó, corrigió, eliminó o agregó filas sin importarlas |
| `ultimaEliminada` | `{ fila, indice, indiceFiltrado } \| null` | Última fila eliminada, para `Deshacer` durante 6 segundos |
| `tourAbierto` / `tourSteps` | `boolean` / `TourStep[]` | Tour guiado de 8 pasos (zona de carga, archivo, resumen, corregir, solo errores, tabla, agregar fila e importar) |

---

## Validaciones

### Del archivo (`LecturaExcelPautaService`)

| Caso | Mensaje |
|---|---|
| No es `.xlsx` / `.xls` | "El archivo no es un Excel" |
| Sin filas debajo de los encabezados | "El archivo no contiene datos" |
| Encabezados distintos a la plantilla: faltantes, no permitidos, repetidos o columnas con datos sin nombre | "El archivo no tiene el formato de la plantilla" + lista de problemas |
| Archivo dañado | "No se pudo leer el archivo" |

Las columnas deben ser exactamente: `Fecha`, `Idtienda`, `Tienda`, `Meta`, `TikTok`, `Google`, `Otros`.

### De cada celda (`ValidacionFilasPautaService`)

| Mensaje en la celda | Cuándo |
|---|---|
| `Obligatorio` | `Fecha`, `Idtienda` o `Meta` vacíos. `TikTok`, `Google` y `Otros` vacíos se importan como `0` |
| `Solo números` | `Idtienda` o un monto con puntos, comas, espacios, letras o signos (no se usan decimales) |
| `Fecha no válida: …` | La fecha no es real (ej. `2026-02-30`) |
| `Tienda no existe` | No hay ninguna tienda con ese `Idtienda` |
| `Tienda inactiva` | La tienda existe pero no está `ACTIVO` (bloquea la importación) |
| `No es tu tienda` | La tienda no está en `sessionStorage.tiendas_asignadas` (con `Todas` se permite cualquiera) |
| `Sin verificar` | Falló la consulta de tiendas; aparece el botón `Reintentar` |

Al pasar el mouse por la celda se ve la explicación completa (`ayudaCelda()`). La columna `Tienda` es de solo lectura: se llena sola con el nombre real de la tienda; si el Excel traía otro nombre, se reemplaza y queda "Antes: …".

---

## Servicios y endpoints

| Servicio                  | Método               | Endpoint                            | Cuándo                        |
| ------------------------- | -------------------- | ----------------------------------- | ----------------------------- |
| `PautaRepository`  | `consultarTiendas()` | `GET metodoGenerico?coleccion=Tienda&Id={ids}&campos=Id,NombreTienda,Estado&mcomp=2` | Al cargar el archivo (una sola consulta con todos los Idtienda) y al cambiar un Idtienda |
| `PautaRepository`  | `insertar()` | `POST metodoGenerico?coleccion=Pauta` | Al confirmar la importación |

---

## Flujo principal

```
onFileChange() / onDrop()
  -> cargarArchivo(): revisa la extensión
  -> leerArchivo()
     -> LecturaExcelPautaService.leerArchivo()
        -> si falla → errorArchivo dentro del modal + temblor
     -> arma FilaPrevia[] y valida cada fila
     -> consultarTiendas(): una sola consulta con todos los Idtienda
        -> mientras tanto la columna Tienda dice "Buscando…"
        -> al responder: llena el nombre real y revalida

Edición en la previsualización
  -> alEditarCelda(): revalida la fila (si cambia el Idtienda borra el nombre)
  -> alSalirDeCelda(): al salir de Idtienda consulta esa tienda
  -> corregirAutomaticamente(): deja solo los dígitos en los errores "Solo números"
  -> eliminarFila() / deshacerEliminar() / agregarFila()

uploadFile()
  -> guarda contra doble clic (isLoading)
  -> sin filas, con tiendas sin verificar o con errores → no importa
  -> convierte Idtienda y montos a número (TikTok, Google y Otros vacíos → 0)
  -> PautaRepository.insertar() → POST a Pauta
  -> Swal éxito → close() → refreshDataEvent.emit()
```

---

## Métodos

### `corregirAutomaticamente()`
Corrige en todas las filas los errores `Solo números` que tengan arreglo: quita todo lo que no sea dígito (`1.500.000` → `1500000`, `$200,000` → `200000`). Como no se usan decimales, `1500.5` queda `15005`. Los valores con letras no se tocan porque no se sabe qué número quisieron poner. Las celdas corregidas quedan en **verde** con "Antes: …" y al final un `Swal` dice cuántas se corrigieron y cuántas filas quedan con errores. El botón solo aparece si hay algo que corregir (`celdasCorregibles`).

### `eliminarFila(fila)` / `deshacerEliminar()`
Quita la fila de lo que se va a importar (no del Excel) y muestra el aviso `Deshacer` durante 6 segundos.

### `agregarFila()`
Agrega una fila al final con la fecha de la última fila (o la de hoy), `TikTok`, `Google` y `Otros` en `0`, y deja el cursor en `Idtienda`. En la columna Fila dice "Nueva".

### `alSalirDeFila(fila, event)`
Con "Solo errores" activo, quita de la lista la fila ya corregida **al salir de ella**, no mientras se escribe.

### `pedirCerrar()`
Lo usan la X, `Cancelar` y Escape. Si hay cambios sin importar (`hayCambios`) pregunta "¿Seguro que quieres salir?" con "Sí, salir" / "Seguir editando" (el foco queda en seguir editando). Solo cargar el archivo sin tocar nada no pregunta.

### `onEscape()`
Escape cierra el modal, salvo mientras se importa, si hay una alerta abierta (ahí Escape cierra la alerta) o si el tour está abierto (ahí solo cierra el tour). `cerrarTour()` baja `tourAbierto` en el siguiente ciclo para que el mismo Escape no cierre también el modal.

---

## Diseño en celular

En pantallas de menos de 768 px (`estilos/_diseno-celular.scss`) el modal sale como **hoja desde abajo**, las filas de la previsualización se ven como tarjetas (cada `td` con su `data-etiqueta`), los campos miden 44 px con letra de 16 px (para que el celular no haga zoom), el aviso `Deshacer` queda pegado abajo y el footer respeta el área segura. En la zona de carga dice "Toca para elegir tu Excel", porque en celular no se puede arrastrar. Con el celular acostado el modal ocupa toda la pantalla.

---

## Historial de cambios

| Fecha      | Autor          | Cambio                                              |
| ---------- | -------------- | --------------------------------------------------- |
| 2026-05-21 | Adalberto González | Creación del componente con importación básica      |
| 2026-05-21 | Adalberto González | Validación de headers obligatorios y filas inválidas |
| 2026-05-21 | Adalberto González | Botón Importar deshabilitado sin archivo seleccionado |
| 2026-10-14 | Yadilton Torres | Nuevo diseño con zona de arrastrar y soltar, tarjeta del archivo y errores dentro del modal con temblor |
| 2026-10-14 | Yadilton Torres | Encabezados exactos de la plantilla (faltantes, no permitidos, repetidos y columnas sin nombre) |
| 2026-10-14 | Yadilton Torres | Previsualización editable con validación por celda, "Solo errores", paginación y "Corregir automáticamente" |
| 2026-10-14 | Yadilton Torres | Verificación de tiendas por Idtienda (no existe, inactiva, no es tu tienda) y nombre de la tienda automático |
| 2026-10-14 | Yadilton Torres | Eliminar fila con Deshacer, Agregar fila y confirmación al cerrar con cambios sin importar |
| 2026-10-14 | Yadilton Torres | Lectura, validación y guardado separados en servicios y repositorio del módulo de Pauta; estilos en `estilos/` |
| 2026-10-14 | Yadilton Torres | Diseño en celular y tour guiado |

---

## Observaciones

> Deuda técnica, comportamientos no obvios, decisiones de diseño que no se ven en el código.

- La normalización de fecha maneja dos formatos de entrada del Excel: serial numérico (tipo `number`) y string `DD/MM/YYYY`. Cualquier otro formato queda como texto y se marca `Fecha no válida`, para que el usuario la escoja en el calendario.
- `rowsMemory` está declarado pero no se usa actualmente — candidato a eliminar en próximo refactor.
- **"No es tu tienda" es solo una validación de frontend** y compara por el nombre de la tienda. La seguridad real debe ir en el backend.
- `(beforeinput)="condicion && metodo()"` no sirve en las celdas: cuando la condición es `false` la expresión devuelve `false` y Angular hace `preventDefault`, así que no deja escribir. Por eso la columna se le pasa al método o se usa la directiva con `[appCampoSoloNumeros]="condicion"`.
- Desde el diálogo "Abrir" de Windows no se puede arrastrar el archivo al modal; hay que arrastrarlo desde el Explorador.
- Volver a importar el mismo Excel **duplica** los registros: todavía no se revisan duplicados por fecha + tienda.
