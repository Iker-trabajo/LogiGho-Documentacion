---

## Autor: Iker Acevedo  
Fecha creacion: 2026-05-05  
Ultima actualizacion: 2026-05-05  
Estado: aceptada

# ADR — Patrón Strategy para nuevas transportadoras en la conciliación de pagos

**Autor:** Iker Acevedo  
**Fecha:** 2026-05-05  
**Estado:** Aceptada

---

## Contexto

`ProcesarConciliacionUseCase` ya gestionaba cuatro transportadoras (Interrapidísimo, Envia, X-Cargo, Servientrega) con lógica hardcodeada: un bloque `else if` por transportadora dentro de `ProcesarLotePedidosSecuencialAsync`, índices individuales con tipos distintos, y métodos de actualización separados (`ActualizarPedidoConPago`, `ActualizarPedidoConPagoEnvia`, etc.).

Al integrar D2E como quinta transportadora, repetir el mismo patrón significaba añadir otro índice, otro `else if`, otro método de actualización, y otro método de lectura en el repositorio — acoplando cada nueva transportadora directamente al use case.

---

## Opciones consideradas

### Opción A — Hardcodear D2E igual que las transportadoras existentes

Añadir `indiceConciliacionPagosD2E`, un bloque `else if` adicional en `ProcesarLotePedidosSecuencialAsync` y un método `ActualizarPedidoConPagoD2E()`.

**Pros:** consistente con lo que ya existe; sin nueva abstracción.  
**Contras:** el use case crece linealmente con cada transportadora nueva; cada adición requiere modificar el orquestador central.

### Opción B — Patrón Strategy para D2E (y futuras transportadoras)

Definir `IConciliacionTransportadora` con tres responsabilidades: nombre de colección (`NombreColeccion`), construcción del índice (`CrearIndice`) y actualización del pedido (`ActualizarPedidoAsync`). El use case itera sobre una lista de estrategias registradas sin conocer los detalles de cada una.

**Pros:** agregar una transportadora nueva implica solo crear una clase nueva sin tocar el use case; separación clara entre orquestación y lógica por transportadora.  
**Contras:** agrega un logica doble dentro del archivo ya que esta la logica acoplada que ya tiene con las demas transportadora y la logica nueva con el Patron Strategy

---

## Decisión

**Se eligió:** Opción B — Patrón Strategy

**Razón:** La plataforma a iniciado a implementar nuevas transportadoras. Hardcodear la quinta haría que la sexta y séptima requirieran editar el use case cada vez. La Strategy desacopla el orquestador de los detalles de cada transportadora y mantiene el punto de extensión claro: añadir una clase que implemente `IConciliacionTransportadora`.

---

## Consecuencias

**Positivas:**

- Incorporar una nueva transportadora no requiere modificar `ProcesarConciliacionUseCase` ni `ProcesarLotePedidosSecuencialAsync`.
- Cada transportadora encapsula sus campos (`NumeroGuia`, `Fecha`, `ValorRecaudar`) en su propia clase.
- La interfaz sirve como contrato documentado para quienes integren futuras transportadoras.
- Se vuelve el proceso mucho mas rapido, ya que para una nueva trasnportadora solo es la instancia y la intefaz configurada a la nueva transportadora

**Negativas:**

- Existen dos enfoques hasta que las legacy se migren: los cuatro `else if` hardcodeados siguen en el use case junto al loop de estrategias.
- La migración de las legacy requiere validación en producción para no romper conciliaciones activas.

---

## Impacto en el código


| Módulo / Archivo                                                      | Cambio                                                                                    |
| --------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| `Aplicacion/Interfaces/Transportadora/IConciliacionTransportadora.cs` | Nueva interfaz con `NombreColeccion`, `CrearIndice`, `ActualizarPedidoAsync`              |
| `Aplicacion/CasosUso/Transportadoras/ConciliacionD2EStrategy.cs`      | Primera implementación de la interfaz para D2E                                            |
| `Aplicacion/CasosUso/Conciliacion/ProcesarConciliacionUseCase.cs`     | Lista `_strategies`; loop que carga índices y procesa estrategias al final de cada pedido |
| `Aplicacion/Interfaces/Repositorio/IConciliacionRepository.cs`        | Nuevo método `ObtenerColeccionAsync(string nombreColeccion)`                              |
| `Infraestructura/Repositorio/ConciliacionRepository.cs`               | Implementación de `ObtenerColeccionAsync`                                                 |


---

## Historial de cambios


| Fecha      | Autor        | Cambio                                        |
| ---------- | ------------ | --------------------------------------------- |
| 2026-05-05 | Iker Acevedo | ADR creado junto con la implementación de D2E |


