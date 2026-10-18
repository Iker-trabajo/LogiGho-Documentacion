---

## Autor:

Fecha creacion: YYYY-MM-DD
Estado: desarrollo | produccion | deprecado
Tipo: vista del Sistema de Experiencias

# Vista: NombreVista

**Clave de vista:** `dominio.vista`
**Ubicación:** `src/app/views/<dominio>/<vista>`
**Ruta:** `/app/<dominio>/<vista>`
**Estándar:** [Crear una vista](../frontend/experiencias/crear-vista.md)

---

## ¿Qué hace?

Qué muestra, para quién y qué problema resuelve.

---

## Versiones por experiencia

| Experiencia | Tiene versión propia | Notas |
|---|---|---|
| `clasica` | Sí (obligatoria) | |
| `<id>` | Sí / No (usa respaldo) | |

---

## Facade

### Lecturas

| Signal | Tipo | Descripción |
|---|---|---|
| `cargando` | `Signal<boolean>` | |

### Acciones

| Método | Descripción |
|---|---|
| `iniciar()` | |

### Permisos

| Permiso | Fuente (backend / archivo temporal) | Controla |
|---|---|---|
| `nombrePermiso` | | |

---

## Repository y endpoints

| Método | Endpoint | Autorización | Filtrado por usuario en servidor |
|---|---|---|---|
| `consultar()` | `GET /ruta` | Cognito | Sí / No (explicar) |

---

## Reglas (`helpers/`)

| Función | Qué calcula |
|---|---|
| `nombreFuncion()` | |

---

## Seguridad

- [ ] Cumple [Seguridad por diseño](../frontend/experiencias/seguridad-por-diseno.md).
- Pendientes conocidos:

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| YYYY-MM-DD | Nombre | Descripción del cambio |

---

## Observaciones

- Deuda técnica, diferencias entre experiencias, pendientes.
