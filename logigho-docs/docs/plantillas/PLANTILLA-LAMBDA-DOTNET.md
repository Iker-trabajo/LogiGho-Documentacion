
## Autor: 
 
Fecha creacion: YYYY-MM-DD  
Estado: desarrollo | produccion | deprecado

--- 
## Lambda: NombreLambda

**Trigger:** API Gateway | EventBridge | SQS  
**AOT:** Sí | No

---

## ¿Qué hace?

2-3 líneas. Qué operación de negocio resuelve.

---

## Accionador


| Método | Ruta              | Auth         |
| ------ | ----------------- | ------------ |
| `POST` | `/api/v1/recurso` | Bearer token |


---

## Request

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

### Exitoso

```json
{
  "campo": "valor"
}
```

### Errores


| Código | Cuándo          |
| ------ | --------------- |
| `400`  | Datos inválidos |
| `401`  | Token inválido  |
| `500`  | Error interno   |


---

## Flujo interno

```
Handler (Aplicacion)
  -> NombreCommand / NombreQuery
     -> INombreRepositorio
        -> MongoDB: nombre_coleccion
```

---

## Dependencias externas

> Completar solo si consume servicios además de MongoDB (S3, SQS, otro Lambda, API externa). Eliminar si no aplica.


| Servicio | Uso         |
| -------- | ----------- |
| `S3`     | Descripción |


---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| YYYY-MM-DD | Nombre | Descripción del cambio |

---

## Observaciones

> Deuda técnica, comportamientos especiales, decisiones no obvias del código.

- Observación

