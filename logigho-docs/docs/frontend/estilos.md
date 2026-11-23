# Estilos y SCSS

!!! info "Estándar propuesto"
    Las reglas de estilos (tokens semánticos, prohibición de hex, temas) están en
    [Sistema de Experiencias: tokens y estilos](experiencias/tokens-y-estilos.md). Mientras la propuesta no se apruebe,
    este documento describe lo que existe hoy: la paleta `--logigho-*` de `src/scss/_theme.scss`.

!!! warning "Tema «La Red» (2026-11-23, verificado en local, pendiente de PreProd)"
    La rama `feature/rediseno-integral-la-red` añade `src/scss/lared/` con los tokens `--lg-*` reales, el modo claro y las utilidades `lg-*`. Lo que existe en el código está en [Sistema visual y tokens](sistema-visual-la-red.md); el estándar de Experiencias de arriba sigue siendo una propuesta y sus nombres de token no coinciden con los del código.

## Estructura SCSS

```
src/
└── scss/
    └── ...
```

## Variables globales

> Documentar variables de color, tipografía, breakpoints.

## Convenciones de estilo

> Documentar convenciones del proyecto.
