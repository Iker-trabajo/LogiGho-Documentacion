---
tipo: deuda-tecnica
prioridad: media
estado: pendiente
fecha_creacion: 2026-09-19
autor: Iker Acevedo
---

# TODO: Integración completa de creación de usuarios API (sin dependencia AWS Cognito)

**Estado:** 🔴 Pendiente (Deuda técnica)

**Prioridad:** Media

**Esfuerzo estimado:** 3-5 días

---

## Problema actual

Hoy, crear un usuario API requiere:

1. **Paso manual:** Crear usuario en AWS Cognito (CLI o UI Cognito)
2. **Paso automático:** UI Angular crea entrada en MongoDB `UsuarioTienda` referenciando el `sub` de Cognito

**Desacoplamiento:**
- Fuente de verdad distribuida: Cognito + MongoDB
- Si `sub` se crea mal en Cognito → usuario API es inútil
- Imposible crear usuarios desde app si Cognito cae
- Admin debe tener acceso a Cognito (escalada de permisos)

---

## Solución objetivo

Integración **end-to-end en el aplicativo** sin tocar Cognito:

```
Frontend AccesosApiComponent
  ↓
POST /api/usuarios-api { sub, email, username, password }
  ↓
Backend Lambda NUEVA: APILambdaCrearUsuarioApi
  ├─ Genera Sub (UUID) único
  ├─ Valida: email no existe en BD
  ├─ Crea entrada en MongoDB `UsuarioTienda`
  ├─ OPCIÓN A: Crea usuario en Cognito via SDK AWS Cognito
  │         (requiere API key, pero sin UI manual)
  ├─ OPCIÓN B: Registra en tabla interna `UsuariosAPI` (sin Cognito)
  │         Genera JWT propio o via Amazon Cognito Resource Owner Password Flow
  └─ Retorna: { sub, token, ... }
  ↓
Frontend recibe credenciales → usuario API listo
```

---

## Opciones de implementación

### Opción A: Lambda + AWS Cognito SDK (Recomendada inicialmente)

Crear usuario en Cognito desde lambda via `AmazonCognitoIdentityProviderClient`.

**Pros:**
- Cognito sigue siendo SSOT (source of truth)
- Usuarios creados automáticamente
- JWT sigue firmado por Cognito

**Contras:**
- Requiere API credentials (access key / secret key)
- Debe gestionarse en secrets de Lambda
- Complejidad: manejo de excepciones Cognito

**Responsable:** Crear tarea NUEVA: `APILambdaCrearUsuarioApi`

### Opción B: MongoDB + JWT interno

No tocar Cognito. Crear tabla `UsuariosAPI` con campos:
- `sub` (UUID generado)
- `email`
- `username`
- `passwordHash` (bcrypt)
- `estado`
- etc.

Lambda genera JWT propio (firma HS256 con secret en env var) o delega a resource owner password flow de Cognito.

**Pros:**
- Independencia de Cognito
- Control total de creación

**Contras:**
- Doble gestión de usuarios (Cognito en dashboard, app en API)
- Más complejidad en autenticación

---

## Tareas desglosadas

### 1. Diseño ADR
- [ ] **Opción A vs B:** decidir arquitectura
- [ ] **Documento:** ADR-003 "Creación de usuarios API sin intervención manual Cognito"

### 2. Backend (Lambda nueva)
- [ ] **Crear proyecto:** `APILambdaCrearUsuarioApi` (siguiendo patrón Clean Architecture)
- [ ] **Interfaz:** `IUsuarioApiServicio`
- [ ] **Implementación:** 
  - Validar input (email, username, password strength)
  - Generar Sub (UUID)
  - Crear entrada MongoDB `UsuarioTienda`
  - [Opción A] Llamar AWS Cognito SDK para crear usuario
  - [Opción B] Hash password, insertar en tabla `UsuariosAPI`
- [ ] **Errores:** 
  - Email duplicado (409 Conflict)
  - Username duplicado (409)
  - Cognito/BD fallan (500)
- [ ] **Tests:** unitarios para validación, integración con MongoDB

### 3. Lambda de autenticación (si Opción B)
- [ ] **Crear proyecto:** `APILambdaLoginUsuarioApi`
- [ ] **Validar:** email + password contra tabla `UsuariosAPI`
- [ ] **Retornar:** JWT firmado (HS256) o token de Cognito (resource owner password)

### 4. Frontend
- [ ] **Agregar modal:** "Crear usuario API" en `AccesosApiComponent`
- [ ] **Formulario adicional:** password (si Opción B) o solo email/username (si Opción A)
- [ ] **Llamar:** `POST /api/usuarios-api` con datos
- [ ] **Feedback:** mostrar `sub` generado, credenciales de login

### 5. Documentación
- [ ] **ADR-003:** decisión arquitectónica
- [ ] **APILambdaCrearUsuarioApi.md:** docs de lambda nueva
- [ ] **APILambdaLoginUsuarioApi.md:** docs de autenticación (si Opción B)
- [ ] **Actualizar:** AccesosApiComponent.md con workflow creación

### 6. Despliegue
- [ ] Crear roles IAM para lambda (si toca Cognito)
- [ ] Secrets Manager para credentials
- [ ] Actualizar template CloudFormation

### 7. Testing e2e
- [ ] Crear usuario → verificar en MongoDB
- [ ] Verificar → login funciona
- [ ] Verificar → API de pedidos valida el nuevo usuario

---

## Impacto esperado

| Métrica | Antes | Después |
| --- | --- | --- |
| Pasos para crear usuario API | 2+ (Cognito + app) | 1 (app) |
| Tiempo estimado | 10+ min | 2 min |
| Dependencia Cognito | 🔴 Sí | 🟡 Parcial (solo auth) |
| Auditoría completa | ❌ No | ✅ Sí (en MongoDB) |

---

## Notas y consideraciones

### Seguridad
- Validar **password strength** si Opción B (mínimo 12 chars, mayúscula, número, símbolo)
- Hash con **bcrypt** (costo 12+) si BD de passwords
- HTTPS obligatorio

### Escalabilidad
- Tabla `UsuariosAPI` (si Opción B): índice en `email`, `username`, `sub`
- Rate limiting en endpoint creación (máx 5 usuarios/minuto por admin)

### Compatibilidad
- Usuarios **existentes** (creados vía Cognito manual) deben seguir funcionando
- No romper `APILambdaConsultarPedidos` durante migración

### Cronograma estimado
- Sprint actual: 3-5 días (paralelizable con otros tasks)
- Recomendación: hacer después de estabilizar cambios actuales (2026-09-25)

---

## Criterios de aceptación

- [ ] Usuario puede crear nuevo usuario API desde UI (sin Cognito manual)
- [ ] Nuevo usuario se refleja en tabla `UsuarioTienda`
- [ ] Nuevo usuario puede autenticarse y consultar pedidos
- [ ] Logs muestran auditoría completa (quién creó, cuándo, cambios)
- [ ] Tests e2e pasan
- [ ] Documentación actualizada

---

## Referencias y enlaces

- [ADR-001: Migración de validación de tiendas](adr-001-migracion-validacion-tiendas.md)
- [AccesosApiComponent.md](../../../frontend/views/administracion/accesos-api/accesos-api.md)
- AWS Cognito Admin SDK: https://docs.aws.amazon.com/AWSJavaScriptSDK/latest/AWS/CognitoIdentityServiceProvider.html
- bcrypt .NET: https://github.com/BcryptNet/bcrypt.net-core

---

## Historial

| Fecha | Autor | Estado |
| --- | --- | --- |
| 2026-09-19 | Iker Acevedo | Documento creado como deuda técnica. Diseño y opciones definidas. |
