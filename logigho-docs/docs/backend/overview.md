# Backend — Overview

El backend de LogiGho está compuesto por **66 proyectos .NET** organizados bajo Clean Architecture,
y un conjunto de **Lambdas Python** para procesamiento serverless.

## Organización

```
LambdasLogiGho/
├── LambdasLogiGho.Aplicacion      → Casos de uso, interfaces, DTOs
├── LambdasLogiGho.Dominio         → Entidades, reglas de negocio, value objects
└── LambdasLogiGho.Infraestructura → Repositorios, acceso a datos, servicios externos
```

## Servicios documentados

| Servicio | Capa | Estado |
|---|---|---|
| [ApiLambdaActualizacionConciliaciones](lambdas-dotnet/lambdas/actualizacion-conciliaciones/actualizacion-conciliaciones.md) | .NET Lambda | desarrollo |
