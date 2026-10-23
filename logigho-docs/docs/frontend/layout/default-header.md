---
autor: Yadilton Torres
fecha_creacion: 2026-10-09
ultima_actualizacion: 2026-10-09
estado: desarrollo
nivel: 1
---

# Barra superior (Default Header)

**Autor:** Yadilton Torres

**Selector:** `app-default-header`

**Ubicación:** `SitioLogiGho/src/app/layout/default-layout/default-header`

---

## ¿Qué hace?

Es la barra blanca de arriba en todas las pantallas internas. Contiene:

- El botón **☰** que abre y cierra el [menú lateral](default-layout.md).
- En celular, el icono del logo, que lleva al muro (`/app/muro-compartido`).
- El botón **"Consultar guía"**, que abre el modal [Consultar Guía](../components/consulta-guia-modal.md).
- Los botones de **notificaciones** (campana con punto rojo), **lista** y **mensajes**.
- La **píldora del usuario**: "Bienvenido" + correo y el avatar con su inicial, que abre el menú de cuenta (perfil, wallet, cerrar sesión, etc.).

Va dentro del layout con `position="sticky"`; la sombra al bajar la pone CoreUI con `cShadowOnScroll`.

---

## Roles y acceso

| Acceso | Descripción |
|---|---|
| Autenticado | Requiere login. Se muestra en todas las rutas bajo `/app` |

---

## Estructura de archivos

```
default-header/
├── default-header.component.ts
├── default-header.component.html
├── default-header.component.scss
└── default-header.component.spec.ts
```

---

## Decoradores y propiedades clave

| Elemento | Nombre | Tipo | Descripción |
|---|---|---|---|
| `@Input` | `sidebarId` | `string` | Id del sidebar que abre el botón ☰ (`sidebar1`) |
| Propiedad | `emailaddress` | `string \| null` | Correo del usuario que se muestra en la píldora |
| Propiedad | `profileInitial` | `string` | Inicial del correo que se muestra en el avatar |
| Propiedad | `consultaGuiaAbierta` | `boolean` | Si el modal "Consultar guía" está abierto |

| Método | Descripción |
|---|---|
| `abrirConsultaGuia()` / `cerrarConsultaGuia()` | Abren y cierran el modal de consulta de guía |
| `logOut()` | Cierra la sesión con `AuthenticateService` (Cognito) |

---

## Diseño

| Elemento | Cómo se ve |
|---|---|
| Barra | Blanca sólida con línea inferior y **56 px** de alto (CoreUI trae 64 px) |
| Botones de avisos | Círculos de **38 px** con fondo lila suave (`rgba(107, 75, 159, 0.08)`), el mismo tono del menú lateral para que se vean como un solo diseño. Al pasar el mouse suben un poco y se oscurece el fondo |
| Campana | Punto rojo de notificación y animación de campana al pasar el mouse |
| Consultar guía | Píldora lila suave con icono propio (hoja + código de barras + lupa) y texto en azul LogiGho. La lupa se anima al pasar el mouse. Anillo de foco solo al llegar con Tab |
| Usuario | Una sola píldora lila con "Bienvenido" + correo y el avatar redondo de **32 px** con degradado naranja |
| Menos movimiento | Se respeta la opción "reducir movimiento" del sistema |

### Tamaños de pantalla

| Pantalla | Comportamiento |
|---|---|
| Escritorio (≥ 1200 px) | Todo visible, "Consultar guía" con texto |
| Portátil pequeño / tablet (< 1200 px) | "Consultar guía" queda como círculo de 38 px con el icono y un tooltip |
| Celular (< 768 px) | ☰ y logo a la izquierda; Consultar guía (solo icono), avisos y avatar a la derecha, con menos espacio entre ellos. El texto de bienvenida se oculta y queda solo el avatar redondo |

> En celular se anula una regla vieja del layout (`estilos/_pagina.scss`) que corría los `c-header-nav` un 3 % a la izquierda, para que todo quede alineado.

---

## Flujo principal

```
Header cargado
  → toma el correo del usuario → emailaddress y profileInitial

Clic en ☰                → abre/cierra el sidebar (escritorio) o el cajón (celular)
Clic en "Consultar guía" → consultaGuiaAbierta = true → <app-consulta-guia-modal>
Cerrar el modal          → cerrarConsultaGuia()
Clic en el avatar        → menú de cuenta → Cerrar sesión → logOut()
```

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-10-09 | Yadilton Torres | Botón "Consultar guía" con su modal (ver [Consultar Guía](../components/consulta-guia-modal.md)). |
| 2026-10-09 | Yadilton Torres | Rediseño: barra blanca de 56 px, botones redondos lila, píldora de usuario con avatar y versión de celular (☰ y logo a la izquierda, acciones a la derecha). Las acciones ahora también se ven en celular. |

---

## Observaciones

- Los botones de lista y mensajes no tienen acción todavía; la campana muestra el punto rojo fijo.
- El texto "Bienvenido + correo" se corta con puntos suspensivos si el correo es muy largo.
