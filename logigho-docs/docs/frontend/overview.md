# Frontend — SitioLogiGho

Aplicación Angular Framework que consume el API Gateway de LogiGho.

!!! info "Tema visual «La Red» (2026-11-23, verificado en local, pendiente de PreProd)"
    La rama `feature/rediseno-integral-la-red` lleva la identidad del login y del registro a toda la plataforma, con modo claro, revisión para celular, animaciones y modo básico. Aún no está en PreProd ni en producción: ver [Rediseño «La Red»](tema-la-red.md).

## Estructura del proyecto

```
SitioLogiGho/
└── src/
    └── app/
        ├── components/      → Componentes reutilizables
        ├── core/            → Servicios, guards, interceptors
        ├── icons/           → Iconografía del sistema
        ├── infrastructure/  → Llamadas HTTP, adaptadores
        ├── layout/          → Estructura visual (navbar, sidebar, etc.)
        └── views/           → Páginas / vistas de la app
```

## Tecnologías

| Herramienta | Uso |
|---|---|
| Angular | Framework principal |
| SCSS | Estilos |
| — | — |
