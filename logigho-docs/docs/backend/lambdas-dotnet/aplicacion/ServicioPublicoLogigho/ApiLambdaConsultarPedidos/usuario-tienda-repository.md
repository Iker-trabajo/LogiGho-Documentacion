---
autor: Iker Acevedo
fecha_creacion: 2026-09-19
estado: produccion
---

# UsuarioTiendaRepository

**Ubicación:** `LambdasLogiGho.Aplicacion/ServicioPublicoLogigho/APILambdaConsultarPedidos/Infrastructura/Repositorio/UsuarioTiendaRepository.cs`

**Interfaz:** `IUsuarioTiendaRepository` (Dominio)

**Responsabilidad:** Acceso a datos de la tabla `UsuarioTienda` en MongoDB. Consulta permisos de usuarios API validando acceso a tiendas.

---

## ¿Qué hace?

Centraliza la lógica de consulta de autorización de usuarios. Reemplaza la lectura de claims JWT personalizados (`custom:idTienda`, `custom:nombreTienda`) con una consulta a la base de datos en cada request.

**Métodos públicos:**
- `static Create()` — Factory que instancia el repositorio con credenciales encriptadas
- `Task<UsuarioTiendaDTO?> ObtenerPermisosPorSubAsync(string sub)` — Busca usuario por Cognito Sub y retorna sus permisos

---

## Arquitectura

### Factory Pattern
```csharp
public static IUsuarioTiendaRepository Create()
{
    string cadenaEncriptada = Environment.GetEnvironmentVariable("CADENA_CONEXION") ?? "";
    string baseDatos = Environment.GetEnvironmentVariable("DATABASE_NAME") ?? "LogighoDB";
    
    var client = new MongoClient(Utils.DesencriptarCadena(cadenaEncriptada));
    var db = client.GetDatabase(baseDatos);
    
    return new UsuarioTiendaRepository(db);
}
```

Inicializa la conexión una sola vez en el constructor estático de `Function`. Reutiliza la instancia para toda la ejecución.

### Inyección de dependencia
```csharp
// En Function.cs
static Function()
{
    _usuarioTiendaRepo = UsuarioTiendaRepository.Create();
}

// En FunctionHandler
var permisosUsuario = await _usuarioTiendaRepo.ObtenerPermisosPorSubAsync(sub);
```

---

## Método: ObtenerPermisosPorSubAsync

### Firma
```csharp
public async Task<UsuarioTiendaDTO?> ObtenerPermisosPorSubAsync(string sub)
```

### Parámetros
| Parámetro | Tipo | Descripción |
| --- | --- | --- |
| `sub` | `string` | Cognito user ID (UUID extraído del JWT) |

### Retorno
```csharp
public class UsuarioTiendaDTO
{
    public string Sub { get; set; }
    public string? Username { get; set; }
    public string? Email { get; set; }
    public bool AccesoATodas { get; set; }
    public List<TiendaInfo> Tiendas { get; set; } = new();
    public bool Activo { get; set; }
}
```

Retorna `null` si usuario no existe en BD.

### Flujo interno
```
ObtenerPermisosPorSubAsync(sub)
  ↓
coleccion.Find(new BsonDocument("Sub", sub))
  → FirstOrDefaultAsync()
  ↓
Si documento encontrado:
  → Deserializa a UsuarioTienda
  → Retorna UsuarioTiendaDTO con datos
Sino:
  → Retorna null
```

### Ejemplo de documento en MongoDB
```json
{
  "_id": ObjectId("67890123456789012345abcd"),
  "Sub": "550e8400-e29b-41d4-a716-446655440000",
  "Username": "servicio_pedidos_api",
  "Email": "api@logigho.com",
  "AccesoATodas": false,
  "Tiendas": [
    { "IdTienda": "156938", "NombreTienda": "Tienda Norte" },
    { "IdTienda": "204710", "NombreTienda": "Tienda Sur" }
  ],
  "Activo": true,
  "FechaCreacion": "2026-06-15T10:30:00Z"
}
```

---

## Validaciones en Function.cs (post-consulta)

Tras obtener `UsuarioTiendaDTO`, `Function.cs` valida:

```csharp
if (permisosUsuario == null || !permisosUsuario.Activo)
{
    Console.WriteLine($"[AUTH] Usuario {sub} no existe en DB o esta inactivo.");
    return Respuesta(headers, HttpStatusCode.Forbidden, true, 
                     "El usuario no tiene permisos asignados o esta inactivo.");
}

// Lógica de AccesoATodas
List<string> idTiendas;
List<string> nombreTiendas;

if (permisosUsuario.AccesoATodas)
{
    // Comodín: sin filtro de tienda
    idTiendas = new List<string> { "*" };
    nombreTiendas = new List<string> { "*" };
    Console.WriteLine($"[AUTH] Usuario {sub} autenticado con Acceso GLOBAL.");
}
else
{
    // Filtro explícito
    idTiendas = permisosUsuario.Tiendas.Select(t => t.IdTienda)
        .Where(id => !string.IsNullOrWhiteSpace(id)).ToList();
    nombreTiendas = permisosUsuario.Tiendas.Select(t => t.NombreTienda ?? "")
        .Where(n => !string.IsNullOrWhiteSpace(n)).ToList();
}
```

---

## Tabla MongoDB: `UsuarioTienda`

### Índices recomendados

```javascript
// Primary lookup: buscar por Sub (Cognito ID)
db.UsuarioTienda.createIndex({ "Sub": 1 })

// Opcional: búsqueda por Email (para UI de gestión)
db.UsuarioTienda.createIndex({ "Email": 1 })

// Opcional: búsqueda por Username
db.UsuarioTienda.createIndex({ "Username": 1 })
```

### Esquema (colección)
| Campo | Tipo | Obligatorio | Descripción |
| --- | --- | --- | --- |
| `_id` | ObjectId | Sí | ID único de MongoDB |
| `Sub` | string | Sí | Cognito user ID (UUID) |
| `Username` | string | Sí | Nombre de usuario API |
| `Email` | string | Sí | Email del usuario |
| `AccesoATodas` | bool | Sí | Si `true`, acceso sin restricción de tiendas |
| `Tiendas` | array | No | Arreglo de { IdTienda, NombreTienda } si AccesoATodas=false |
| `Activo` | bool | Sí | Control de activación/desactivación sin borrar |
| `FechaCreacion` | string (ISO 8601) | No | Fecha de creación (metadato) |

---

## Manejo de errores

### Conexión fallida
```csharp
// Si MongoDB no responde:
// → MongoClient lanza MongoConnectionException
// → Lambda no la atrapa, propaga a APIGateway
// → Retorna 500 Internal Server Error
```

### Usuario no encontrado
```csharp
// FirstOrDefaultAsync() retorna null
// Function.cs detecta: if (permisosUsuario == null)
// → Retorna 403 Forbidden "El usuario no tiene permisos asignados"
```

### Usuario inactivo
```csharp
// permisosUsuario.Activo == false
// Function.cs detecta: if (!permisosUsuario.Activo)
// → Retorna 403 Forbidden "usuario está inactivo"
```

---

## Impacto de performance

### Latencia añadida
- **Consulta MongoDB:** ~1-2 ms (preprod Atlas)
- **Deserialization:** < 0.5 ms
- **Total:** ~2 ms por request

Negligible vs. timeout de 30s de la lambda.

### Recomendación
- Crear índice en `Sub` (ver Índices arriba)
- Monitorear CloudWatch Logs si latencia crece

---

## Migración desde Claims JWT

### Antes (2026-06-24)
```
JWT claims → JwtClaimsHelper.ExtraerClaimsUsuario()
  → custom:idTienda, custom:nombreTienda
  → Pasadas a ConsultarPedidosUseCase
```

### Ahora (2026-09-19)
```
JWT claims → JwtClaimsHelper.ExtraerIdentidadUsuario()
  → sub, email, username
  → UsuarioTiendaRepository.ObtenerPermisosPorSubAsync(sub)
  → Retorna tiendas desde BD
  → Pasadas a ConsultarPedidosUseCase
```

**Resultado:** Validación inmediata, auditable, flexible.

---

## Historial de cambios

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-09-19 | Iker Acevedo | Repositorio creado. Factory pattern. Consulta MongoDB `UsuarioTienda` por Sub. |
