# Diagrama del sistema

## Arquitectura general

```
[SitioLogiGho — Angular]
          |
     [API Gateway]
          |
    ┌─────┴─────┐
    |           |
[.NET Lambdas]  [Python Lambdas]
    |
[Clean Architecture]
  Aplicacion
  Dominio
  Infraestructura
```

## Flujo de una petición típica

1. El usuario interactúa con SitioLogiGho (Angular)
2. Angular consume el API Gateway
3. El API Gateway enruta hacia el servicio correspondiente
4. El servicio procesa y responde
5. Angular renderiza el resultado
