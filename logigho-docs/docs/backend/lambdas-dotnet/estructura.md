# Estructura Clean Architecture — .NET

## Capas

### Aplicacion
Contiene los casos de uso del sistema. Orquesta el flujo entre Dominio e Infraestructura.
- Comandos y Queries (CQRS si aplica)
- DTOs
- Interfaces de servicios

### Dominio
El corazón del sistema. No depende de ninguna capa externa.
- Entidades
- Value Objects
- Reglas de negocio
- Eventos de dominio

### Infraestructura
Implementaciones concretas de las interfaces definidas en Aplicacion.
- Repositorios
- Acceso a base de datos
- Servicios externos (email, storage, etc.)
- Configuración

## Dependencias entre capas

```
Infraestructura → Aplicacion → Dominio
```

Dominio no conoce a nadie. Aplicacion conoce Dominio. Infraestructura conoce a ambas.
