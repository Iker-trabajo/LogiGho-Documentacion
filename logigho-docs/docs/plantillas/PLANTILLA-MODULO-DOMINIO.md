---

## Autor:   
Fecha creacion: YYYY-MM-DD  
Estado: desarrollo | produccion | deprecado

# Dominio: NombreModulo

**Namespace:** `LambdasLogiGho.Dominio.NombreModulo`

---

## ¿Qué representa?

2-3 líneas. Qué concepto de negocio modela este módulo y por qué existe como entidad propia.

---

## Entidad principal: `NombreEntidad`


| Propiedad   | Tipo     | Descripción         |
| ----------- | -------- | ------------------- |
| `Id`        | `Guid`   | Identificador único |
| `Propiedad` | `string` | Descripción         |


> Omitir propiedades cuyo nombre ya las explica. Solo documentar las que requieren contexto de negocio.

### Reglas de negocio


| Regla              | Descripción                                  |
| ------------------ | -------------------------------------------- |
| Nombre de la regla | Qué valida y qué excepción lanza si se viola |


---

## Interfaces de repositorio


| Método                   | Retorno                | Descripción         |
| ------------------------ | ---------------------- | ------------------- |
| `ObtenerPorId(Guid id)`  | `Task<NombreEntidad?>` | Busca por ID        |
| `Guardar(NombreEntidad)` | `Task`                 | Persiste la entidad |


---

## Relaciones con otros módulos

> Eliminar si no tiene dependencias con otros módulos del dominio.


| Módulo       | Relación                                                     |
| ------------ | ------------------------------------------------------------ |
| `OtroModulo` | `NombreEntidad` referencia a `OtroModulo` por `OtroModuloId` |


---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| YYYY-MM-DD | Nombre | Descripción del cambio |

---

## Observaciones

> Decisiones de diseño no obvias, deuda técnica, restricciones del modelo.

- Observación

