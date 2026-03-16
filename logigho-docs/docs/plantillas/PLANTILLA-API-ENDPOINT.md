---
autor: 
fecha_creacion: YYYY-MM-DD
ultima_actualizacion: YYYY-MM-DD
estado: desarrollo | produccion | deprecado
---

# Endpoint: METODO /ruta

**Autor:** Nombre Apellido  
**Servicio destino:** Nombre del servicio .NET o Lambda que lo resuelve  
**Autenticación:** Requerida | No requerida  
**Roles permitidos:** Todos | admin | vendedor | (el que aplique)

---

## ¿Qué hace?

Descripción breve en 1-2 líneas de qué resuelve este endpoint.

---

## Request

### Headers

| Header | Valor | Requerido |
|---|---|---|
| `Authorization` | `Bearer {token}` | Sí |
| `Content-Type` | `application/json` | Sí |

### Parámetros de URL

> Solo si la ruta tiene parámetros. Ej: `/api/v1/tarjetas/{id}`

| Parámetro | Tipo | Descripción |
|---|---|---|
| `id` | `Guid` | ID del recurso |

### Query params

> Solo si aplica. Ej: `/api/v1/tarjetas?pagina=1&limite=10`

| Parámetro | Tipo | Requerido | Default | Descripción |
|---|---|---|---|---|
| `pagina` | `int` | No | `1` | Número de página |
| `limite` | `int` | No | `10` | Resultados por página |

### Body

```json
{
  "campo": "string",
  "otrocampo": 0
}
```

| Campo | Tipo | Requerido | Descripción |
|---|---|---|---|
| `campo` | `string` | Sí | Descripción del campo |
| `otrocamp` | `int` | No | Descripción del campo |

---

## Response

### Respuesta exitosa

**Código:** `200 OK` | `201 Created` | `204 No Content`

```json
{
  "id": "guid",
  "campo": "valor",
  "fechaCreacion": "2026-01-01T00:00:00Z"
}
```

| Campo | Tipo | Descripción |
|---|---|---|
| `id` | `Guid` | ID del recurso creado |
| `campo` | `string` | Descripción |

### Respuestas de error

| Código HTTP | Código interno | Cuándo ocurre |
|---|---|---|
| `400` | `BAD_REQUEST` | Datos del body inválidos |
| `401` | `UNAUTHORIZED` | Token inválido o expirado |
| `403` | `FORBIDDEN` | Sin permisos para esta acción |
| `404` | `NOT_FOUND` | Recurso no existe |
| `500` | `INTERNAL_ERROR` | Error inesperado del servidor |

**Formato de error:**
```json
{
  "error": {
    "code": "CODIGO_ERROR",
    "message": "Descripción legible del error"
  }
}
```

---

## Ejemplo completo

### Request
```http
POST /api/v1/tarjetas
Authorization: Bearer eyJhbGc...
Content-Type: application/json

{
  "numero": "4111111111111111",
  "titular": "Juan Perez"
}
```

### Response
```http
HTTP/1.1 201 Created

{
  "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "titular": "Juan Perez",
  "fechaCreacion": "2026-03-15T10:00:00Z"
}
```

---

## Flujo interno

```
API Gateway
  -> Valida token JWT
  -> Enruta a [NombreServicio]
  -> [NombreCommand / NombreQuery]
     -> [NombreRepositorio]
        -> Base de datos
  -> Retorna respuesta
```

---

## Notas adicionales

> Limitaciones, rate limiting, comportamientos especiales, casos borde.

---

## Changelog

| Fecha | Autor | Cambio |
|---|---|---|
| YYYY-MM-DD | Nombre | Descripción |
