---
autor: Iker Acevedo
fecha_creacion: 2026-09-19
estado: aceptada
---

# ADR-001 — Validación de tiendas: Claims JWT → Base de datos

**Autor:** Iker Acevedo  
**Fecha:** 2026-09-19  
**Estado:** Aceptada

---

## Contexto

Originalmente, `APILambdaConsultarPedidos` extraía las tiendas autorizadas de los **claims personalizados del JWT** (`custom:idTienda` y `custom:nombreTienda`). Cada vez que un usuario Cognito cambiaba de tiendas autorizadas, había que:

1. Modificar el JWT en AWS Cognito
2. Esperar a que el cliente refrescara su sesión
3. Confiar en que Cognito y la lambda estuvieran sincronizados

**Problemas:**
- **Desacoplamiento:** La fuente de verdad estaba distribuida entre Cognito y la lógica de la lambda
- **Latencia:** Cambios en tiendas tardaban hasta que el cliente refrescaba el token (20s en LogiGho)
- **Auditoría:** Modificaciones de tiendas no quedaban registradas en MongoDB
- **Flexibilidad:** Un usuario con acceso a 100 tiendas saturaba el JWT con claims enormes

---

## Opciones consideradas

### Opción A — Mantener claims JWT (Original)

Validar tiendas exclusivamente desde el JWT.

**Pros:**
- Cero cambios en código
- Validación stateless en la lambda
- Compatible con caché de JWT

**Contras:**
- Latencia de sincronización (hasta 20s)
- Sin registro de cambios en BD
- JWT con claims enormes para usuarios con muchas tiendas
- Imposible revocar acceso inmediatamente

### Opción B — Validación en BD (Elegida)

Una tabla MongoDB `UsuarioTienda` centraliza la verdad sobre quién accede a qué. La lambda consulta esta tabla en cada request.

**Pros:**
- Validación inmediata: cambios en BD aplican al siguiente request
- Registro auditable: todos los cambios en MongoDB
- Escalable: usuario con 100 tiendas no infla el JWT
- Flexible: permite lógica compleja (`AccesoATodas`, permisos por rol, etc.)
- UI para gestionar accesos sin tocar Cognito

**Contras:**
- Latencia adicional: una consulta MongoDB por request (ms, no crítico)
- Requiere tabla `UsuarioTienda` en production

### Opción C — Hibridación (Claims + BD con caché)

Mantener claims para caché y validar contra BD cada N requests.

**Pros:**
- Reduce consultas MongoDB

**Contras:**
- Complejo de debuggear
- Riesgo de inconsistencia entre claims y BD
- Ventaja marginal (MongoDB es rápido en preprod/prod)

---

## Decisión

**Se eligió:** Opción B — **Validación en BD**

**Razón:** 
Centraliza la fuente de verdad en MongoDB. Cambios aplican inmediatamente. Permite UI para gestión sin tocar Cognito. Costo de latencia (1-2 ms por request en Atlas) es negligible vs. beneficios de auditabilidad e inmediatez.

---

## Consecuencias

**Positivas:**
- ✅ Revocación inmediata de acceso (sin esperar refresco de token)
- ✅ Auditoría completa en MongoDB de quién accede a qué
- ✅ Escalabilidad: sin límite en cantidad de tiendas por usuario
- ✅ UI de gestión (`AccesosApiComponent`) sin dependencia de Cognito
- ✅ Lógica flexible: `AccesoATodas`, permisos por rol, inactivación de usuarios

**Negativas:**
- ⚠️ Latencia adicional: +1-2ms por request (MongoDB Atlas en preprod/prod)
- ⚠️ Complejidad: tabla adicional que mantener en sincronía
- ⚠️ Risk: si MongoDB baja, lambda falla (mitigado: usar réplicas, timeouts)

---

## Impacto en el código

| Módulo / Repo | Cambio |
| --- | --- |
| `LambdasLogiGho` | `APILambdaConsultarPedidos/Function.cs`: Antes extraía tiendas de claims JWT. Ahora consulta `UsuarioTiendaRepository.ObtenerPermisosPorSubAsync(sub)`. |
| `LambdasLogiGho` | Nueva clase: `UsuarioTienda` (dominio). Nueva interface: `IUsuarioTiendaRepository` (dominio). Nueva implementación: `UsuarioTiendaRepository` (infraestructura). |
| `SitioLogiGho` | Nueva vista: `AccesosApiComponent` en `views/administracion/configuracion-api`. Permite CRUD de usuarios API y asignación de tiendas sin escribir código. |

---

## Flujo de autorización (Post-migración)

```
Cliente envía GET /pedidos?...
  ↓
Lambda.FunctionHandler()
  → Extrae JWT del header
  → Decodifica claims: extrae 'sub' (Cognito user ID)
  → UsuarioTiendaRepository.ObtenerPermisosPorSubAsync(sub)
       → Consulta MongoDB tabla 'UsuarioTienda' por Sub
       → Retorna { Sub, Username, Email, AccesoATodas, Tiendas[], Activo }
  → Valida: ¿usuario activo? ¿tiene tiendas?
  → Si AccesoATodas=true → comodín "*" (todas las tiendas)
  → Si no → lista explícita de tiendas
  → ConsultarPedidosUseCase filtra por tiendas autorizadas
  ↓
Respuesta con pedidos solo de tiendas autorizadas en BD
```

---

## Referencias

- Tabla MongoDB: `UsuarioTienda` (colección en BD LogighoDB)
- Documentación actualizada: `ApiLambdaConsultarPedidos.md` (v2)
- Vista de gestión: `AccesosApiComponent.md`
- Repositorio: `UsuarioTiendaRepository.md`

---

## Historial

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-09-19 | Iker Acevedo | ADR creado. Migración de claims JWT → BD implementada. |
