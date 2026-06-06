---

## Autor: Adalberto González
Fecha creacion: 2026-06-06
Estado: produccion

# Servicio: AuthenticateService

**Ubicación:** `src/app/core/application/services/cognito/cognito-service.service.ts`  
**Scope:** `providedIn: 'root'`

---

## ¿Qué hace?

Este servicio se encarga de gestionar el acceso de los usuarios a la aplicación. Controla procesos como el inicio de sesión, el cierre de sesión, la renovación automática de credenciales y la validación de que la sesión siga siendo válida durante el uso de la plataforma.

---

## Métodos

### `login(emailaddress: string, password: string): Promise<boolean>`

Autentica al usuario en Cognito y almacena `id_token`, `accessToken` y `refresh_token` en storage.

**Retorna:** `true` si el login fue exitoso, `false` en caso contrario.

---

### `logOut(): void`

Cierra la sesión en Cognito, limpia sessionStorage y el caché de paginación, y redirige al login.

---

### `refreshToken(): Promise<boolean>`

Renueva el `id_token` usando el `refresh_token` guardado en localStorage.

**Retorna:** `true` si el refresco fue exitoso.

---

### `isAuthenticated(): boolean`

Verifica si existe un `id_token` activo en sessionStorage.

**Retorna:** `true` si el usuario tiene sesión activa.

---

### `getAuthToken(): Observable<boolean>`

Verifica autenticación y redirige al login si no hay sesión. Usado como guard en rutas protegidas.

---

### `getUserRole(): string | null`

Retorna el rol del usuario desde sessionStorage.

---

## Endpoints que consume

| Método | Ruta                  | Descripción          |
| ------ | --------------------- | -------------------- |
| `POST` | `insertarAuditoria`   | Registra auditoría de login exitoso y fallido |

---

## Historial de cambios

| Fecha      | Autor              | Cambio |
|------------|--------------------|--------|
| 2026-06-06 | Adalberto González | Se agregó `PaginationCacheService` al constructor y se llama `clearCache()` en `logOut()` para limpiar IndexedDB al cambiar de cuenta. |

---

## Observaciones