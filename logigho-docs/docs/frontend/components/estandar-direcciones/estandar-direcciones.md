---

## Autor: Adalberto González

Fecha creacion: 2026-05-26  
Estado: desarrollo  
Tipo: componente

# Componente: DireccionEstandarizadaComponent

**Selector:** `app-direccion-estandarizada`  
**Ubicación:** `src/app/components/direccion-estandarizada/`  
**Acceso:** Público (Todos)

---

## ¿Qué hace?

Es una ventana flotante que ayuda a los usuarios a convertir y saber si una dirección esta bien escrita, segun la nomenclatura colombiana (urbana o rural).

---

## Decoradores

| Decorador | Nombre          | Tipo                   | Descripción                                               |
| --------- | --------------- | ---------------------- | --------------------------------------------------------- |
| `@Output` | `cerrar`        | `EventEmitter<void>`   | Se emite cuando el usuario cierra el modal                |
| `@Output` | `direccionLista`| `EventEmitter<string>` | Se emite con la dirección estandarizada al pulsar "Usar"  |

---

## Propiedades clave

| Propiedad          | Tipo                                   | Descripción                                                                 |
| ------------------ | -------------------------------------- | --------------------------------------------------------------------------- |
| `modo`             | `Signal<'libre' \| 'constructor'>`     | Controla qué panel se muestra al usuario                                    |
| `parsedAddress`    | `Signal<Partial<ParsedAddress> \| null>` | Resultado del parser; alimenta la sección de resultado y el desglose       |
| `form`             | `FormDir`                              | Objeto mutable con todos los campos del constructor; tiene index signature para el binding dinámico de complementos |
| `desgloseFilas`    | `Signal<{clave, valor}[]>`             | Filas del desglose; bifurca entre campos urbanos y rurales según `parsedAddress().zona` |
| `colorConf`        | `Signal<string>`                       | Color CSS según confianza: verde ≥80, naranja ≥50, rojo <50                |
| `tipoGenActivo`    | `Signal<boolean>`                      | Habilita el selector de tipo de vía generadora en el constructor            |
| `masAbierto`       | `Signal<boolean>`                      | Muestra/oculta los complementos extra (bloque, casa, interior…)             |
| `libre$`           | `Subject<string>`                      | Canal RxJS con `debounceTime(300)` para no parsear en cada tecla            |

---

## Servicios y endpoints

| Servicio              | Método             | Endpoint | Cuándo                                                    |
| --------------------- | ------------------ | -------- | --------------------------------------------------------- |
| `AddressParserService`| `parse(raw)`       | —        | En cada cambio de texto libre (con debounce de 300 ms)    |
| `AddressParserService`| `buildAddress(f)`  | —        | En cada cambio de campo en el constructor                 |

---

## Flujo principal

```
Usuario abre modal desde floating-bar
  -> modo = 'libre' (default)

Modo libre:
  onInputLibre(v)
    -> libre$.next(v)
    -> debounce 300ms
    -> parser.parse(v)
    -> parsedAddress.set(resultado)
    -> desglose / confianza / barra se actualizan por computed()

Modo constructor:
  onConstructorChange()
    -> parser.buildAddress(form)  [construye string]
    -> parsedAddress.set({ ...form, direccionEstandarizada, confianza })

Ambos modos:
  usarDireccion()
    -> direccionLista.emit(direccionEstandarizada)
    -> cerrar.emit()
```

---

## Historial de cambios

| Fecha      | Autor              | Cambio                    |
| ---------- | ------------------ | ------------------------- |
| 2026-05-26 | Adalberto González | Creación del módulo       |

---

## Observaciones

> Deuda técnica, comportamientos no obvios, decisiones de diseño que no se ven en el código.

- Hay cierto tipo de direccines que no es estan contempladas en el manual, lo cual puede dar con un resultado diferente.
