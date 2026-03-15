# Manejo de errores

## Formato estándar de error

```json
{
  "error": {
    "code": "CODIGO_ERROR",
    "message": "Descripción del error",
    "details": []
  }
}
```

## Códigos de error del sistema

| Código HTTP | Código interno | Descripción |
|---|---|---|
| 400 | BAD_REQUEST | — |
| 401 | UNAUTHORIZED | — |
| 403 | FORBIDDEN | — |
| 404 | NOT_FOUND | — |
| 500 | INTERNAL_ERROR | — |
