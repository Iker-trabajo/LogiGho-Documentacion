---

## Autor:   
Fecha creacion: YYYY-MM-DD  
Estado: desarrollo | produccion | deprecado

# Endpoint: MÉTODO /ruta

**Lambda / Servicio:** NombreServicio  
**Auth:** Requerida | No requerida  
**Rol:** admin | vendedor | todos

---

## ¿Qué hace?

1-2 líneas.

---

## Request

**Headers:** `Authorization: Bearer {token}` · `Content-Type: application/json`

### Parámetros de URL *(eliminar si no aplica)*


| Parámetro | Tipo   | Descripción    |
| --------- | ------ | -------------- |
| `id`      | `Guid` | ID del recurso |


### Body *(eliminar si es GET)*

```json
{
  "campo": "string"
}
```


| Campo   | Tipo     | Requerido | Descripción |
| ------- | -------- | --------- | ----------- |
| `campo` | `string` | Sí        | Descripción |


---

## Response

### Exitoso `200 OK`

```json
{
  "campo": "valor"
}
```

### Errores


| Código | Cuándo                        |
| ------ | ----------------------------- |
| `400`  | Datos del body inválidos      |
| `401`  | Token inválido o expirado     |
| `403`  | Sin permisos para esta acción |
| `404`  | Recurso no existe             |
| `500`  | Error inesperado del servidor |


---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| YYYY-MM-DD | Nombre | Descripción del cambio |

---

## Observaciones

> Limitaciones, rate limiting, comportamientos especiales, casos borde.

- Observación

