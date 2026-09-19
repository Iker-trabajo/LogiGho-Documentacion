---
autor: Iker Acevedo
fecha_creacion: 2026-09-19
estado: desarrollo
---

# Vista: Gestión de Accesos API (`AccesosApiComponent`)

**Selector:** `app-accesos-api`

**Ubicación:** `src/app/views/administracion/configuracion-api/accesos-api`

**Módulo:** `administracion`

---

## ¿Qué hace?

Permite a administradores gestionar usuarios API y sus permisos de acceso a tiendas. Antes, los permisos viajaban en claims JWT de Cognito; ahora se centralizan en MongoDB tabla `UsuarioTienda`, permitiendo:

- ✅ CRUD de usuarios API (crear, editar, activar/desactivar)
- ✅ Asignación/revocación de tiendas por usuario (drag-and-drop o doble lista)
- ✅ Lógica "Acceso Global" (`AccesoATodas`) para usuarios con acceso a todas las tiendas
- ✅ Gestión dinámica de campos API (`ConfiguracionCamposApi`) sin redeploy
- ✅ Preview en tiempo real del JSON que devuelve la API

---

## Estructura del directorio

```
configuracion-api/
├── accesos-api/
│   ├── accesos-api.component.ts          ← Lógica (signals, computed, métodos)
│   ├── accesos-api.component.html        ← Template (tabs, doble lista, formulario)
│   ├── accesos-api.component.scss        ← Estilos (badges, cards, drag-drop)
│   ├── accesos-api.component.spec.ts     ← Tests unitarios
│   ├── helpers/
│   │   └── accesos-api.repository.ts     ← Llamadas a API backend
│   └── models/
│       └── usuario-tienda.interface.ts   ← Interfaces TypeScript
└── routes.ts                             ← Rutas del módulo
```

---

## Propiedades y Estados (Signals)

### Carga de datos
| Señal | Tipo | Descripción |
| --- | --- | --- |
| `cargando` | `signal(bool)` | True mientras se cargan usuarios, tiendas, campos |
| `guardando` | `signal(bool)` | True mientras se guarda cambios |
| `error` | `signal(string \| null)` | Mensaje de error si algo falla |

### Usuarios API
| Señal | Tipo | Descripción |
| --- | --- | --- |
| `_usuariosApi` | `signal(UsuarioTienda[])` | Lista completa de usuarios API de BD |
| `usuarioSeleccionado` | `signal(UsuarioTienda \| null)` | Usuario siendo editado en el formulario |
| `modoNuevoUsuario` | `signal(bool)` | True si estamos creando un usuario nuevo |
| `usuariosFiltrados` (computed) | `UsuarioTienda[]` | Filtrado por búsqueda (nombre, email, sub) |

### Tiendas
| Señal | Tipo | Descripción |
| --- | --- | --- |
| `_tiendasCatalogo` | `signal(TiendaCatalogo[])` | Todas las tiendas activas de la BD |
| `tiendasAsignadas` | `signal(TiendaInfo[])` | Tiendas asignadas al usuario actual |
| `tiendasSeleccionadasIds` | `signal(Set<string>)` | IDs de tiendas seleccionadas para asignar (dual list) |
| `tiendasDisponiblesPaginadas` (computed) | `TiendaCatalogo[]` | Tiendas NO asignadas, paginadas (20 por página) |
| `tiendasAsignadasFiltradas` (computed) | `TiendaInfo[]` | Tiendas asignadas filtradas por búsqueda |

### Formulario
| Propiedad | Tipo | Descripción |
| --- | --- | --- |
| `accesosForm` | `FormGroup` | Reactive form: sub, username, email, accesoATodas, activo |

### Campos API
| Señal | Tipo | Descripción |
| --- | --- | --- |
| `_camposApi` | `signal(any[])` | Campos dinámicos de la API (`ConfiguracionCamposApi`) |
| `camposApiPaginados` (computed) | `any[]` | Campos con pagination (10 por página) |
| `jsonPreview` (computed) | `string` | JSON de ejemplo que devuelve la API |

---

## Métodos principales

### Carga inicial
| Método | Descripción |
| --- | --- |
| `ngOnInit()` | Inicializa form y carga datos |
| `private cargarDatos()` | Promise.all: usuarios, tiendas, campos desde repositorio |
| `private initForm()` | Crea FormGroup con validadores (Cognito sub, username, email) |

### Selección de usuario
| Método | Descripción |
| --- | --- |
| `seleccionarUsuario(u: UsuarioTienda)` | Carga usuario en el form, captura snapshot de tiendas asignadas |
| `iniciarNuevoUsuario()` | Limpia form para crear usuario nuevo |
| `descartarCambios()` | Revierte form al estado original |

### Gestión de tiendas (doble lista)
| Método | Descripción |
| --- | --- |
| `seleccionarTiendaCheck(t)` | Toggle: agrega/quita tienda de seleccionadas |
| `asignarSeleccionadas()` | Mueve tiendas seleccionadas de "disponibles" a "asignadas" |
| `removerTienda(id)` | Quita tienda específica de asignadas |
| `removerTodas()` | Quita todas las tiendas (con confirmación Swal) |
| `drop(event: CdkDragDrop)` | Maneja drag-and-drop entre listas |

### Guardado
| Método | Descripción |
| --- | --- |
| `async guardarAccesos()` | POST/PUT según modo (crear o editar). Valida form. Recarga datos. |

### Campos API
| Método | Descripción |
| --- | --- |
| `async crearCampo()` | Modal Swal para crear campo nuevo en ConfiguracionCamposApi |
| `async toggleEstadoCampo(c)` | Toggle Activo/Inactivo de un campo |

---

## Modelos e Interfaces

### UsuarioTienda (Backend)
```typescript
interface UsuarioTienda {
  _id?: string;
  Sub: string;              // Cognito user ID (UUID)
  Username: string;
  Email: string;
  AccesoATodas: boolean;    // Si true, acceso a TODAS las tiendas
  Tiendas: TiendaInfo[];    // Lista explícita si AccesoATodas=false
  Activo: boolean;
  FechaCreacion?: string;   // ISO 8601
}

interface TiendaInfo {
  IdTienda: string;
  NombreTienda?: string;
}
```

### TiendaCatalogo (de API)
```typescript
interface TiendaCatalogo {
  Id: string;
  NombreTienda: string;
  Ecosistema?: string;      // "propia", "proveedor", "dropshipping"
  Activo: boolean;
}
```

---

## Flujo de datos

### Carga inicial
```
ngOnInit()
  → initForm()
  → cargarDatos()
       → Promise.all([
           getUsuariosApi(),
           getTiendasActivas(),
           getCamposApi()
         ])
       → _usuariosApi.set(usuarios)
       → _tiendasCatalogo.set(tiendas)
       → _camposApi.set(campos)
       → Si hay usuarios: seleccionarUsuario(primero)
       → Si no: iniciarNuevoUsuario()
```

### Asignación de tiendas
```
Usuario selecciona tiendas en lista "disponibles"
  ↓
asignarSeleccionadas()
  → Filtra: no duplicadas en tiendasAsignadas
  → Agrega: tiendasAsignadas.set([...actual, ...nuevas])
  → Limpia: tiendasSeleccionadasIds.set(new Set())
  ↓
Frontend renderiza lista "asignadas" actualizada
```

### Guardado
```
guardarAccesos()
  → Valida form (sub, username, email son requeridos)
  → Valida: al menos 1 tienda O AccesoATodas=true
  → Si modo nuevo:
       → POST /api/accesos-usuarios { Sub, Username, Email, AccesoATodas, Tiendas, Activo }
  → Si modo edición:
       → PUT /api/accesos-usuarios/{_id} { Username, Email, AccesoATodas, Tiendas, Activo }
  → Recarga: cargarDatos()
  → Selecciona: usuario recién guardado en lista
```

---

## Patrones de validación

### Sub (Cognito ID)
Regex: `^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$` (UUID hex)

### Username
Regex: `^[a-zA-Z0-9._-]{3,50}$` (alfanuméricos, punto, guion, guion bajo)

### Email
Validador nativo `Validators.email`

### Snapshot de cambios
Antes de editar, se captura:
- `tiendasIdsOriginales`: Set de IDs de tiendas al cargar usuario
- `accesoATodasOriginal`: valor de AccesoATodas al cargar

Permite detectar cambios y mostrar footer "Cambios pendientes".

---

## Gestión de cambios (footer sticky)

La señal `cambiosPendientes` (computed) cuenta:
1. ¿Nuevo usuario? → 1 cambio
2. ¿AccesoATodas cambió? → +1
3. ¿Activo cambió? → +1
4. ¿Tiendas añadidas? → +1 por cada nueva
5. ¿Tiendas removidas? → +1 por cada removida

Si `cambiosPendientes > 0` → muestra footer con botones "Guardar" / "Descartar".

---

## Componentes CoreUI usados

| Componente | Uso |
| --- | --- |
| `CardModule` | Tarjetas de layout |
| `TabsComponent` | Tabs: "Usuarios", "Tiendas", "Campos" |
| `TableModule` | Tablas de usuarios y campos |
| `BadgeModule` | Badges de estado y tipo de tienda |
| `FormModule` | Inputs del formulario |
| `ButtonModule` | Botones de acción |
| `GridModule` | Layout grid responsive |
| `AlertModule` | Mensajes de error/éxito |

---

## Librerías externas

| Librería | Uso |
| --- | --- |
| `@angular/cdk/drag-drop` | Drag-and-drop de tiendas |
| `@angular/forms` | Reactive forms |
| `sweetalert2` | Modales de confirmación y creación |

---

## Responsividad

- **Desktop (≥ lg):** Doble lista side-by-side (disponibles ← → asignadas)
- **Mobile (< lg):** Tabs entre "Disponibles" / "Asignadas" (`vistaMovilTiendas` signal)

---

## Cambios visuales vs. versión anterior

| Aspecto | Antes | Ahora |
| --- | --- | --- |
| Fuente de tiendas | Claims JWT (editable en Cognito) | MongoDB tabla `UsuarioTienda` |
| Gestión de permisos | Manual en Cognito | UI completa en Angular |
| Asignación de tiendas | N/A | Doble lista drag-and-drop |
| Acceso global | N/A | Toggle `AccesoATodas` |
| Gestión de campos | N/A | CRUD en UI |
| Preview JSON | N/A | JSON dinámico actualizado |

---

## Historial de cambios

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-09-19 | Iker Acevedo | Componente creado. Migración de gestión de tiendas de Cognito a MongoDB BD. UI completa con doble lista, drag-drop, CRUD de campos. |
