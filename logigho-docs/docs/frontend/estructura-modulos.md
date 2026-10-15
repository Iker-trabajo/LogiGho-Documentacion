# Estructura de módulos

## app/components
Componentes reutilizables en toda la aplicación. Sin lógica de negocio.

> Documentar componentes a medida que se desarrollen.

## app/core
Servicios singleton, guards de rutas, interceptors HTTP.

> Documentar servicios core a medida que se desarrollen.

## app/infrastructure
Capa de acceso a datos del frontend. Servicios que consumen el API Gateway.

> Documentar servicios de infraestructura a medida que se desarrollen.

## app/layout
Componentes de estructura visual: navbar, sidebar, footer.

- [Menú lateral (Default Layout)](layout/default-layout.md): menú lateral, buscador del menú, uso con el teclado, cajón del celular y la directiva `SidebarInteraccionDirective`.

> Documentar la barra superior, el footer y los demás elementos del layout a medida que se desarrollen.

## app/views
Páginas completas de la aplicación. Cada vista corresponde a una ruta.

> Documentar vistas a medida que se desarrollen.
