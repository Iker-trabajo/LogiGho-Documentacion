---
autor: 
fecha_creacion: YYYY-MM-DD
ultima_actualizacion: YYYY-MM-DD
estado: desarrollo | produccion | deprecado
nivel: 1 | 2 | 3
---

# Módulo de Dominio: NombreModulo

**Autor:** Nombre Apellido  
**Namespace:** `LambdasLogiGho.Dominio.NombreModulo`  
**Capa:** Dominio — sin dependencias externas

---

## ¿Qué representa?

Descripción en 2-3 líneas. Qué concepto del negocio modela este módulo.
Ejemplo: "Representa una tarjeta de pago registrada por un usuario de LogiGho.
Contiene las reglas de negocio sobre validaciones, estados y límites de una tarjeta."

---

## Entidades

### `NombreEntidad`

Descripción breve de qué es esta entidad en términos del negocio.

#### Propiedades

| Propiedad | Tipo | Requerido | Descripción |
|---|---|---|---|
| `Id` | `Guid` | Sí | Identificador único |
| `Propiedad` | `string` | Sí | Descripción |
| `PropiedadOpcional` | `string?` | No | Descripción |
| `FechaCreacion` | `DateTime` | Sí | Cuándo fue creada |

#### Reglas de negocio

> Las reglas que esta entidad hace cumplir. Si una regla se viola, qué excepción lanza.

| Regla | Descripción | Excepción |
|---|---|---|
| Regla 1 | El campo X no puede ser vacío | `NombreExcepcion` |
| Regla 2 | El valor Y debe ser mayor a 0 | `NombreExcepcion` |

#### Métodos de dominio

##### `NombreMetodo(param: tipo): tipo`
**Descripción:** Qué hace este método en términos de negocio.

**Parámetros:**
- `param` — Descripción

**Proceso:**
1. Valida que...
2. Aplica la regla de...
3. Retorna...

**Retorna:** Descripción

---

## Value Objects

### `NombreValueObject`

Descripción de qué representa y por qué es un Value Object (inmutable, sin identidad propia).

#### Propiedades

| Propiedad | Tipo | Descripción |
|---|---|---|
| `Valor` | `string` | El valor que encapsula |

#### Validaciones

| Validación | Descripción |
|---|---|
| Formato | Debe cumplir el formato X |
| Longitud | Mínimo X, máximo Y caracteres |

---

## Enumeraciones

### `NombreEnum`

| Valor | Descripción |
|---|---|
| `Estado1` | Qué significa este estado |
| `Estado2` | Qué significa este estado |

---

## Interfaces de repositorio

### `INombreRepositorio`

Define el contrato que Infraestructura debe implementar.

| Método | Retorno | Descripción |
|---|---|---|
| `ObtenerPorId(Guid id)` | `Task<NombreEntidad?>` | Busca por ID |
| `Guardar(NombreEntidad)` | `Task` | Persiste la entidad |
| `Eliminar(Guid id)` | `Task` | Elimina por ID |
| `ObtenerTodos()` | `Task<IList<NombreEntidad>>` | Retorna todos |

---

## Excepciones de dominio

| Excepción | Cuándo se lanza |
|---|---|
| `NombreNotFoundException` | Cuando no se encuentra el recurso por ID |
| `NombreValidationException` | Cuando una regla de negocio es violada |

---

## Relaciones con otros módulos

| Módulo | Tipo de relación | Descripción |
|---|---|---|
| `OtroModulo` | Referencia por ID | NombreEntidad tiene un OtroModuloId |

---

## Eventos de dominio

> Solo si el módulo emite eventos. Eliminar sección si no aplica.

### `NombreEvent`

**Cuándo se emite:** Descripción del momento en que ocurre.

**Datos del evento:**
| Campo | Tipo | Descripción |
|---|---|---|
| `Id` | `Guid` | ID de la entidad afectada |
| `Fecha` | `DateTime` | Cuándo ocurrió |

---

## Changelog del módulo

| Fecha | Autor | Cambio |
|---|---|---|
| YYYY-MM-DD | Nombre | Descripción |

---

## Observaciones

> Decisiones de diseño, deuda técnica, cosas no obvias del modelo.

- Observación 1
