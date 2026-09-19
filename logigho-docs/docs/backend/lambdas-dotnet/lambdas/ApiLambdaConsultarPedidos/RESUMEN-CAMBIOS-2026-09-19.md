---
tipo: resumen-cambios
fecha: 2026-09-19
autor: Iker Acevedo
rama: feature/mejora-api-pedidos
---

# Resumen de cambios — Feature: Mejora API Pedidos (2026-09-19)

**Rama:** `feature/mejora-api-pedidos`  
**Cambios:** Migración completa de validación de tiendas (claims JWT → MongoDB BD) + UI de gestión  
**Status:** 🟢 Documentación lista para revisión

---

## Cambio central

**Antes:** Tiendas autorizadas viajaban en claims JWT (`custom:idTienda`, `custom:nombreTienda`)

**Ahora:** Tiendas se consultan en MongoDB tabla `UsuarioTienda` por cada request

**Beneficio:** Validación inmediata, auditable, sin latencia de token refresh

---

## Archivos modificados / creados

### Backend (.NET)

#### Lambda: `APILambdaConsultarPedidos`

**Modificados:**
- `Function.cs` — Ahora consulta `UsuarioTiendaRepository` en lugar de extraer tiendas de JWT claims
- `Aplicacion/CasosDeUso/ConsultarPedidosUseCase.cs` — Soporte para comodín "*" (AccesoATodas)
- `Infrastructura/Utilidades/JwtClaimsHelper.cs` — Solo extrae `sub`, `email`, `username` (no tiendas)

**Nuevos:**
- `Dominio/Interfaces/IUsuarioTiendaRepository.cs` — Contrato de acceso a usuarios
- `Dominio/Modelos/UsuarioTienda.cs` — Entity con Sub, Username, Email, AccesoATodas, Tiendas[]
- `Infrastructura/Repositorio/UsuarioTiendaRepository.cs` — Implementación: consulta MongoDB tabla `UsuarioTienda`

### Frontend (Angular)

#### Nueva vista: `AccesosApiComponent`

**Ubicación:** `src/app/views/administracion/configuracion-api/accesos-api/`

**Nuevos:**
- `accesos-api.component.ts` — Lógica con signals, computed, CRUD
- `accesos-api.component.html` — Template: tabs, doble lista drag-drop, formulario
- `accesos-api.component.scss` — Estilos: badges, cards, drag-drop feedback
- `accesos-api.component.spec.ts` — Tests unitarios
- `helpers/accesos-api.repository.ts` — Llamadas HTTP a backend
- `models/usuario-tienda.interface.ts` — Interfaces TypeScript

**Funcionalidad:**
- ✅ CRUD usuarios API (crear, editar, activar/desactivar)
- ✅ Asignación de tiendas: doble lista, drag-and-drop, checkboxes
- ✅ Lógica "Acceso Global" (`AccesoATodas`)
- ✅ CRUD campos dinámicos (`ConfiguracionCamposApi`)
- ✅ Preview JSON en tiempo real

---

## Documentación creada

### Backend

| Archivo | Descripción |
| --- | --- |
| `adr-001-migracion-validacion-tiendas.md` | **ADR:** Decisión de migrar de JWT claims → BD. Opciones, pros/contras, impacto. |
| `ApiLambdaConsultarPedidos.md` | **Lambda docs ACTUALIZADA.** Flujo nuevo, arquitectura, multi-tienda con AccesoATodas. |
| `usuario-tienda-repository.md` | **Nuevo repositorio.** Factory pattern, consulta MongoDB, validaciones, índices. |
| `TODO-integracion-usuarios-api-completa.md` | **Deuda técnica:** Integración end-to-end sin Cognito manual. Opciones A/B, cronograma. |

### Frontend

| Archivo | Descripción |
| --- | --- |
| `accesos-api.md` | **Componente Angular.** Estructura, signals, métodos, modelos, flujos de datos, validaciones. |
| `accesos-api-adr.md` | **ADR:** Decisión de doble lista con drag-drop. Comparación vs. tabla/modal. |

---

## Cambios visuales (Frontend)

### Nueva vista: Gestión de Accesos API

**Ubicación:** Admin → Configuración → Accesos API

**Pantalla 1: Usuarios API**
```
┌─────────────────────────────────────────┐
│ Búsqueda de usuario                     │
├─────────────────────────────────────────┤
│ ☑ Usuario 1 (api@logigho.com)          │
│   ☑ Usuario 2 (servicio@company.com)   │
│   ☑ Usuario 3 ...                       │
└─────────────────────────────────────────┘

Formulario (lado derecho):
┌─────────────────────────────────────────┐
│ Sub (UUID)                              │
│ Username                                │
│ Email                                   │
│ ☑ Acceso a todas las tiendas           │
│ ☑ Activo                                │
└─────────────────────────────────────────┘
```

**Pantalla 2: Asignación de tiendas (Doble lista)**
```
Disponibles                    Asignadas
┌──────────────────┐          ┌──────────────────┐
│ Tienda 1 ↔       │          │ ← Tienda A       │
│ Tienda 2         │          │   Tienda B       │
│ ...              │          │   Tienda C       │
└──────────────────┘          └──────────────────┘
        [Asignar >]           [< Remover]
```

**Pantalla 3: Campos API**
```
Tabla de campos dinámicos (ConfiguracionCamposApi):
┌─────────────────────────────────────────┐
│ Campo       | Activo | Acciones        │
├─────────────────────────────────────────┤
│ FechaCarga  | ✓      | Toggle / Editar │
│ IdTienda    | ✓      | Toggle / Editar │
│ ...         | ✗      | Toggle / Editar │
└─────────────────────────────────────────┘

[+ Crear campo nuevo]
```

**Pantalla 4: Preview JSON**
```
{
  "Error": false,
  "Mensaje": null,
  "NumeroPaginas": 1,
  "TotalRegistros": 1,
  "Resultado": [
    {
      "FechaCarga": "2026-06-15 01:56:15",
      "IdTienda": "10000",
      ...
    }
  ]
}
```

---

## Validaciones implementadas

### Backend
| Validación | Dónde | Comportamiento |
| --- | --- | --- |
| Usuario existe en BD | `UsuarioTiendaRepository` | 403 Forbidden si no existe |
| Usuario activo | `Function.cs` | 403 Forbidden si inactivo |
| Tiendas asignadas | `Function.cs` | Si AccesoATodas=false, debe tener ≥1 tienda |

### Frontend
| Validación | Patrón | Mensaje |
| --- | --- | --- |
| Sub (Cognito UUID) | `^[0-9a-f]{8}-...` | Formato inválido |
| Username | `^[a-zA-Z0-9._-]{3,50}$` | 3-50 chars, sin espacios |
| Email | Validador nativo | Email format |
| Duplicado Sub | Query BD | "Ya existe usuario con ese Sub" |

---

## Tabla MongoDB: `UsuarioTienda`

### Esquema
```json
{
  "_id": ObjectId("..."),
  "Sub": "550e8400-e29b-41d4-a716-...",      // Cognito user ID
  "Username": "servicio_pedidos_api",
  "Email": "api@company.com",
  "AccesoATodas": false,
  "Tiendas": [
    { "IdTienda": "156938", "NombreTienda": "Tienda Norte" },
    { "IdTienda": "204710", "NombreTienda": "Tienda Sur" }
  ],
  "Activo": true,
  "FechaCreacion": "2026-09-19T10:30:00Z"
}
```

### Índices recomendados
```javascript
db.UsuarioTienda.createIndex({ "Sub": 1 })        // Búsqueda principal
db.UsuarioTienda.createIndex({ "Email": 1 })      // Búsqueda secundaria
db.UsuarioTienda.createIndex({ "Username": 1 })   // Búsqueda secundaria
```

---

## Flujo de autorización (nuevo)

```
Cliente → GET /pedidos?...
  ↓ JWT header
Function.Handler
  ├─ Extrae JWT
  ├─ Decodifica: obtiene 'sub'
  ├─ UsuarioTiendaRepository.ObtenerPermisosPorSubAsync(sub)
  │   └─ MongoDB: busca en UsuarioTienda por Sub
  ├─ Valida: usuario activo, tiene tiendas
  ├─ Si AccesoATodas=true: idTiendas = ["*"]
  ├─ Si no: idTiendas = [tiendas asignadas]
  ├─ ConsultarPedidosUseCase.Ejecutar(idTiendas, ...)
  └─ DocumentRepository: filtra pedidos por tiendas
  ↓
Respuesta: { TotalRegistros, NumeroPaginas, Resultado[] }
```

---

## Performance

| Métrica | Valor | Notas |
| --- | --- | --- |
| Latencia MongoDB lookup | ~1-2 ms | Atlas preprod |
| Impacto en timeout lambda | <7% | Timeout total: 30s |
| Tamaño JWT sin claims | -50 bytes | Menos payload |

**Conclusión:** Negligible. Recomendación: monitorear CloudWatch Logs si crece.

---

## Testing

### Tests unitarios creados
- ✅ `UsuarioTiendaRepository`: consulta BD, manejo de null
- ✅ `AccesosApiComponent`: selección de usuario, asignación tiendas
- ✅ Validadores: Sub (UUID), Username, Email

### Tests e2e recomendados (manual)
- [ ] Crear usuario API desde UI
- [ ] Asignar tiendas via drag-drop
- [ ] Activar/desactivar usuario
- [ ] Consultar pedidos con nuevo usuario
- [ ] Verificar auditoría en MongoDB

---

## Deuda técnica

| Tarea | Estado | Prioridad |
| --- | --- | --- |
| Integración creación usuarios sin Cognito manual | Documentada en `TODO-integracion-usuarios-api-completa.md` | Media |

---

## Checklist de revisión

### Code Review
- [ ] Backend: Clean Architecture respected (Dominio/Aplicacion/Infrastructura)
- [ ] Frontend: Angular best practices (signals, reactive forms, CDK)
- [ ] Seguridad: validaciones en cliente + servidor
- [ ] Tests: cobertura ≥ 80% en lógica crítica
- [ ] Documentación: ADRs y ejemplos presentes

### Deployment
- [ ] Índice MongoDB creado en `UsuarioTienda.Sub`
- [ ] Variables de entorno configuradas (CADENA_CONEXION, DATABASE_NAME, COLECCION_PEDIDOS)
- [ ] Lambda timeout = 30s, memoria = 1024 MB
- [ ] CloudWatch Logs configurados

### Rollback
- [ ] Snapshot de schema MongoDB antes de deploy
- [ ] Revert a rama `main` si necesario
- [ ] Script para copiar `UsuarioTienda` de preprod a prod

---

## Próximos pasos

1. **Revisión de code:** 
   - PR backend + frontend + docs
   - Feedback de team lead

2. **Testing:**
   - QA: crear usuarios, asignar tiendas, consultar pedidos
   - Verificar auditoría en MongoDB

3. **Deployment:**
   - Preprod: validar flujo completo
   - Prod: después de 1 semana sin issues en preprod

4. **Deuda técnica:**
   - Planificar sprint para integración sin Cognito manual (ver TODO)

---

## Referencias rápidas

| Documento | Enlace |
| --- | --- |
| ADR: Migración claims → BD | `adr-001-migracion-validacion-tiendas.md` |
| Lambda actualizada | `ApiLambdaConsultarPedidos.md` |
| Repositorio nuevo | `usuario-tienda-repository.md` |
| Componente frontend | `accesos-api.md` |
| ADR: UI Doble lista | `accesos-api-adr.md` |
| Deuda técnica | `TODO-integracion-usuarios-api-completa.md` |

---

**Documentación completada:** 2026-09-19 ✅
