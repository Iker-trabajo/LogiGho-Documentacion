---

## Autor: Adalberto González

Fecha creacion: 2026-05-26  
Ultima actualizacion: 2026-10-15  
Estado: desarrollo  
Tipo: componente

# Componente: DireccionEstandarizadaComponent

**Selector:** `app-direccion-estandarizada`  
**Ubicación:** `src/app/components/direccion-estandarizada/`  
**Acceso:** Público (Todos)

---

## ¿Qué hace?

Es un modal flotante que ayuda a los usuarios a estandarizar y saber si una dirección esta bien escrita, segun la nomenclatura colombiana (urbana o rural). Ofrece dos modos: texto libre con parsing automático en tiempo real, y constructor guiado campo por campo. Al confirmar, emite la cadena estandarizada para que el componente padre la use o la copie al portapapeles.

En el **Constructor** la persona primero elige, con dos tarjetas con ejemplos, cómo es la dirección:

- **Dirección numerada:** lleva el símbolo `#` (ej. `Calle 45 # 7 - 20`). Se arma con vía, generadora, placa, cuadrante y complementos.
- **Manzana, barrio o vereda:** no lleva `#` (ej. `Mz 8 Lt 2 Casa 4`, `Vereda La Esperanza`). Se arma con el tipo y nombre del lugar, manzana, lote, casa o km.

Lo abren la barra flotante (📍, solo copia la dirección), [Modal creación de pedidos](modal-creacion-pedidos.component.md) (llena el campo Dirección) y la vista de flujo masivo de guías.

---

## Estructura de archivos

```
direccion-estandarizada/
├── direccion-estandarizada.component.ts
├── direccion-estandarizada.component.html
├── direccion-estandarizada.component.scss
├── direccion-estandarizada.component.spec.ts   → 13 pruebas
├── direccion-estandar.service.ts               → Lector y armado de la dirección (ver Estandarizador de direcciones)
└── direccion-estandar.service.spec.ts          → 17 pruebas
```

| Pieza | Para qué la usa el modal |
|---|---|
| [Estandarizador de direcciones](../core/estandarizador-direcciones.md) | Leer el texto libre, armar la dirección, avisos y confianza |
| [Tour guiado](guided-tour.md) | Botón **?** del encabezado |

---

## Decoradores

| Decorador | Nombre          | Tipo                   | Descripción                                               |
| --------- | --------------- | ---------------------- | --------------------------------------------------------- |
| `@Output` | `cerrar`        | `EventEmitter<void>`   | Se emite cuando el usuario cierra el modal                |
| `@Output` | `direccionLista`| `EventEmitter<string>` | Se emite con la dirección estandarizada al pulsar "Usar"  |
| `@HostListener` | `document:keydown.escape` | — | Escape cierra una ventana a la vez (ver `onEscape()`) |
| `@HostListener` | `document:keydown` | — | Con el aviso de salida abierto, las flechas pasan el foco entre "Salir" y "Seguir editando" |

---

## Propiedades clave

| Propiedad          | Tipo                                   | Descripción                                                                 |
| ------------------ | -------------------------------------- | --------------------------------------------------------------------------- |
| `modo`             | `Signal<'libre' \| 'constructor'>`     | Controla qué panel se muestra al usuario                                    |
| `tipoDireccion`    | `Signal<'numerada' \| 'sector'>`       | Tarjeta elegida en el Constructor: dirección numerada o manzana, barrio o vereda |
| `parsedAddress`    | `Signal<Partial<estandarDireccion> \| null>` | Resultado del parser; alimenta la sección de resultado y el desglose  |
| `form`             | `FormDir`                              | Objeto mutable con todos los campos del constructor; tiene index signature para el binding dinámico de complementos |
| `desgloseFilas`    | `Signal<{clave, valor}[]>`             | Filas del desglose; bifurca entre campos urbanos, rurales, de conjunto y de lugar según `parsedAddress().zona` |
| `colorConf`        | `Signal<string>`                       | Color CSS según confianza: verde ≥80, naranja ≥50, rojo <50                |
| `tipoGenActivo`    | `Signal<boolean>`                      | Habilita el selector de tipo de vía generadora en el constructor            |
| `masAbierto`       | `Signal<boolean>`                      | Muestra/oculta los complementos extra (bloque, casa, manzana, lote…)        |
| `confirmarSalida`  | `Signal<boolean>`                      | Muestra el aviso "¿Salir?" cuando hay datos escritos                        |
| `tourAbierto`      | `Signal<boolean>` / `tourSteps`        | Estado y pasos (12) del tour guiado                                         |
| `libre$`           | `Subject<string>`                      | Canal RxJS con `debounceTime(300)` para no parsear en cada tecla            |

Constantes del archivo:

| Constante | Contenido |
|---|---|
| `TIPOS_DIRECCION` | Las dos tarjetas: título, pista y 2 ejemplos de cada una |
| `COMPS_FIJOS` / `COMPS_EXTRA` | Complementos de la dirección numerada (los extra van en "Más complementos") |
| `COMPS_SECTOR` | Manzana, Lote, Casa y Kilómetro de la dirección por lugar |
| `TIPOS_LUGAR` | Viene del servicio: Barrio, Urbanización, Conjunto, Sector, Vereda, Corregimiento, Finca, Parcelación, Batallón |

---

## Tipos de dirección en el Constructor

La regla para elegir es una sola: **¿la dirección lleva `#`?** Si sí, es numerada; si no, es de manzana, barrio o vereda. Por eso `Calle 3 Manzana 1 Casa 6` (calle sin `#`) va en la segunda tarjeta, con la calle en **Calle de referencia**.

| | Dirección numerada | Manzana, barrio o vereda |
|---|---|---|
| Campos | Tipo de vía, número, letra, `#`, generadora, `-`, placa, cuadrante, complementos, barrio (opcional) | Tipo de lugar, **nombre del lugar (obligatorio)**, manzana, lote, casa, km, calle de referencia, referencia |
| Ejemplo de resultado | `CL 45 # 7 - 20 AP 301` | `MZ 8 LT 2 CASA 4 BARRIO BELÉN` · `VEREDA LA ESPERANZA CASA 5` · `FINCA EL RECREO KM 3` |
| Avisos | — | Falta el nombre del lugar · Falta al menos un número (manzana, lote, casa o km) |
| Confianza | Vía 20 + número 25 + generadora 25 + placa 20 + completa 10 | Lugar 40 + un número 40 + sin avisos 20 |

Las dos tarjetas guardan lo escrito: si la persona cambia de tarjeta y vuelve, sus datos siguen ahí. **Limpiar campos** vuelve a "Dirección numerada".

---

## Servicios y endpoints

| Servicio              | Método             | Endpoint | Cuándo                                                    |
| --------------------- | ------------------ | -------- | --------------------------------------------------------- |
| `DireccionEstandarizadaService`| `parse(raw)`       | —        | En cada cambio de texto libre (con debounce de 300 ms)    |
| `DireccionEstandarizadaService`| `buildAddress(f)`  | —        | En cada cambio de campo en el constructor (`zona: 'urbana'` o `'sector'`) |
| `DireccionEstandarizadaService`| `advertenciasSector(f)` / `calcConfianzaSector(f)` | — | En el constructor, con la tarjeta "Manzana, barrio o vereda" |

---

## Flujo principal

```
Usuario abre modal desde floating-bar, modal de pedidos o flujo masivo
  -> modo = 'libre' (default)

Modo libre:
  onInputLibre(v)
    -> libre$.next(v)
    -> debounce 300ms
    -> parser.parse(v)
    -> parsedAddress.set(resultado)
    -> desglose / confianza / barra se actualizan por computed()

Modo constructor:
  setTipoDireccion('numerada' | 'sector')   [tarjetas]
  onConstructorChange()
    -> numerada: parser.buildAddress({ ...form, zona: 'urbana' })  + calcConfianzaForm()
    -> sector:   parser.buildAddress({ ...form, zona: 'sector' })  + advertenciasSector() + calcConfianzaSector()
    -> parsedAddress.set({ ...form, direccionEstandarizada, confianza, advertencias })

Ambos modos:
  usarDireccion()
    -> direccionLista.emit(direccionEstandarizada)
    -> cerrar.emit()

Cerrar (X, Cancelar, clic fuera, Escape):
  intentarCerrar()
    -> si hay datos escritos: aviso "¿Salir?" (Salir / Seguir editando)
    -> si no: cerrar.emit()
```

---

## Teclado y tour

- **Escape** cierra una ventana a la vez, de la más reciente a la más vieja: primero el tour, luego el aviso de salida y por último el modal.
- El aviso de salida es propio del modal (no Swal: Swal queda por debajo del modal). Abre con el foco en **Seguir editando** y las flechas pasan entre los dos botones.
- El botón **?** del encabezado abre el tour. Tiene pasos de los dos modos y de las dos tarjetas; el tour salta los que no están en pantalla. No se muestra en celular.

---

## Diseño celular

| Ancho | Qué cambia |
|---|---|
| Computador | Ventana de 880 px, tarjetas lado a lado con 2 ejemplos cada una |
| ≤ 640 px | Tarjetas una debajo de otra con un solo ejemplo, campos de lugar en una columna, campos a 16 px (evita el zoom de iPhone), fórmula en dos líneas y botones a lo ancho |
| ≤ 480 px | Modal a pantalla completa |
| ≤ 380 px | Tipo de vía en su propia línea |

---

## Historial de cambios

| Fecha      | Autor              | Cambio                    |
| ---------- | ------------------ | ------------------------- |
| 2026-05-26 | Adalberto González | Creación del módulo       |
| 2026-10-15 | Yadilton Torres    | Tarjetas "Dirección numerada / Manzana, barrio o vereda" con ejemplos, dirección por lugar (tipo y nombre del lugar, manzana, lote, casa, km), manzana en el texto libre, tour guiado, Escape en orden, aviso al salir con datos, nuevos estilos y diseño celular |

---

## Observaciones

> Deuda técnica, comportamientos no obvios, decisiones de diseño que no se ven en el código.

- Hay cierto tipo de direccines que no es estan contempladas en el manual, lo cual puede dar con un resultado diferente.
- Las tarjetas siguen los dos tipos de dirección de la guía del formulario de **Base Ventas** ("numerada" y "por sector/rural"); en el modal la segunda se llama **Manzana, barrio o vereda** para que cualquiera la entienda.
- Inter Rapidísimo sí acepta direcciones que empiezan por manzana y no tienen calle: se comprobó con una guía real creada con `MZ H # LOTE 37 - . URBANIZACION VILLAS DEL MOMPOX`.
- **Pendiente:** los campos de texto (nombre del lugar, referencia, calle de referencia y barrio) no quitan los símbolos que el estándar no permite (`( ) / ! , .`), ni las tildes. Inter solo permite `#` y `-`, y Base Ventas rechaza `( ) /`.
- "Usar dirección" no bloquea aunque haya avisos: el aviso y la confianza baja le dicen a la persona qué falta.
