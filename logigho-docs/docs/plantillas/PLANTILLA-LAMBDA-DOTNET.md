---
autor: 
fecha_creacion: YYYY-MM-DD
ultima_actualizacion: YYYY-MM-DD
estado: desarrollo | produccion | deprecado
nivel: 1 | 2 | 3
---

# Nombre del Servicio / Lambda

**Autor:** Nombre Apellido  
**Solución:** `LambdasLogiGho`  
**Namespace:** `LambdasLogiGho.NombreServicio`

---

## ¿Qué hace?

Descripción breve en 2-3 líneas.

---

## Estructura Clean Architecture

```
NombreServicio/
├── Aplicacion/
│   ├── Commands/
│   │   └── NombreCommand.cs
│   ├── Queries/
│   │   └── NombreQuery.cs
│   └── DTOs/
│       └── NombreDto.cs
├── Dominio/
│   ├── Entidades/
│   │   └── NombreEntidad.cs
│   └── Interfaces/
│       └── INombreRepositorio.cs
└── Infraestructura/
    └── Repositorios/
        └── NombreRepositorio.cs
```

---

## Capa Dominio

### Entidades

#### `NombreEntidad`
Descripción de la entidad y sus responsabilidades.

| Propiedad | Tipo | Descripción |
|---|---|---|
| `Id` | `Guid` | Identificador único |
| `Propiedad` | `string` | Descripción |

### Reglas de negocio
- Regla 1
- Regla 2

---

## Capa Aplicacion

### Commands

#### `NombreCommand`
**Descripción:** Qué hace este command.

**Parámetros de entrada:**
| Campo | Tipo | Requerido | Descripción |
|---|---|---|---|
| `Campo` | `string` | Sí | Descripción |

**Proceso:**
1. Paso 1
2. Paso 2

**Retorna:** `NombreDto`

### Queries

#### `NombreQuery`
**Descripción:** Qué consulta este query.

**Parámetros:**
| Campo | Tipo | Descripción |
|---|---|---|
| `Id` | `Guid` | ID a consultar |

**Retorna:** `NombreDto`

---

## Capa Infraestructura

### `NombreRepositorio`
Implementación de `INombreRepositorio`.

| Método | Descripción |
|---|---|
| `ObtenerPorId(Guid id)` | Busca por ID en BD |
| `Guardar(NombreEntidad)` | Persiste la entidad |

---

## Endpoints expuestos

| Método | Ruta | Command/Query | Descripción |
|---|---|---|---|
| `POST` | `/api/v1/recurso` | `NombreCommand` | Crea un recurso |
| `GET` | `/api/v1/recurso/{id}` | `NombreQuery` | Obtiene por ID |

---

## Dependencias externas

| Librería / Servicio | Versión | Uso |
|---|---|---|
| Entity Framework | — | ORM |
| — | — | — |

---

## Base de datos

| Tabla | Descripción |
|---|---|
| `nombre_tabla` | Almacena los registros de este módulo |

---

## Changelog del servicio

| Fecha | Autor | Cambio |
|---|---|---|
| YYYY-MM-DD | Nombre | Descripción |

---

## Observaciones

> Deuda técnica, pendientes, decisiones que no son obvias.

- Observación 1
