# Correos de autenticación (plantillas para SES)

Diseño de los correos que envía el dominio de autenticación. Cada correo es una **guía de envío**, con la forma de la de una transportadora, sobre el azul noche de la página de registro:
- Papel blanco con el logo.
- Ruta TU TIENDA ➜ CLIENTES.
- Número de guía con código de barras.
- Recuadros DE / PARA.
- Código con botón de copiar y botón principal, dentro de la guía.
- Casillas de datos.

Los archivos (HTML, texto plano, imágenes y vistas previas) se entregaron al equipo de backend en la carpeta `plantillas-correo-logigho`, con su `LEEME-para-backend.md`. Esta página resume lo que hay que saber. El contrato de las rutas está en [API de autenticación](../api/autenticacion.md).

---

## Los cuatro correos

| Plantilla | Cuándo se envía | Asunto |
|---|---|---|
| `01-registro-codigo` | `POST /autenticacion/registro/solicitudes` y su reenvío | Tu guía de activación de LogiGho |
| `02-registro-cuenta-existente` | Solicitud de registro con un correo que ya tiene cuenta | Tu correo ya tiene su lugar en LogiGho |
| `03-recuperacion-codigo` | `POST /autenticacion/recuperacion/solicitudes` | Tu guía de seguridad de LogiGho |
| `04-contrasena-cambiada` | Después de `POST /autenticacion/recuperacion/confirmaciones` exitoso | Tu cuenta de LogiGho quedó protegida |

**Título del registro:** "{{nombre}}, tu ecommerce ya juega en las grandes ligas y con el mejor equipo". Debajo van tres beneficios: 1.103 municipios, 5 transportadoras y 1 plataforma.

---

## Variables

Usan la forma `{{variable}}`, la misma de las plantillas de SES (`CreateTemplate` / `SendTemplatedEmail`).

| Variable | Qué va | Correos |
|---|---|---|
| `{{logo_tinta_url}}` | URL pública HTTPS de `logo-tinta.png` | Todos |
| `{{img_barras_url}}` | URL pública HTTPS de `codigo-barras.png` | Todos |
| `{{nombre}}` | `userName` del registro (si no se tiene, quitarlo de la frase) | 01, 03, 04 |
| `{{codigo_1}}` … `{{codigo_6}}` | Cada dígito del código, uno por casilla | 01, 03 |
| `{{codigo}}` | El código completo (botón Copiar, número de guía `LG-…` y texto plano) | 01, 03 |
| `{{minutos}}` | Vigencia del código (15) | 01, 03 |
| `{{enlace}}` | `URL_REGISTRO?solicitud=<id>` o `URL_RECUPERACION?solicitud=<id>` | 01, 03 |
| `{{url_login}}` / `{{url_recuperar}}` | `https://www.logigho.com/app/login` / `.../app/recuperar` | 02, 04 |
| `{{fecha}}` | Fecha y hora del cambio en hora de Colombia, corta (`5 nov 2026 · 1:06 p. m.`) | 04 |

Si un valor trae texto escrito por la persona (como `{{nombre}}`), se escapa para HTML antes de ponerlo.

---

## Reglas del diseño

- **HTML para correo:** tablas y estilos en línea, sin JavaScript ni hojas externas. Los fondos van también con `bgcolor` para Outlook.
- **Imágenes opcionales:** el correo se entiende aunque no carguen; el logo muestra el texto "LogiGho".
- **Código de barras real:** es Code 128 y codifica `LOGIGHO.COM`. Es el mismo para todos los correos.
- **El botón "Copiar" no copia solo:** los correos no permiten programación, así que no hay forma de copiar al portapapeles con un clic. Muestra el código con `user-select: all`, de modo que un toque lo selecciona completo. Gmail en Android además ofrece su propio "Copiar código".
- **Seguridad:** ningún correo dice si una cuenta existe, salvo el 02, que solo llega al dueño del correo. El enlace no lleva el código ni el correo, solo `?solicitud=`.
- **Texto plano:** cada HTML tiene su `.txt`. Conviene enviar las dos partes para entregar mejor y no caer en spam.

---

## Pendientes

- [ ] Subir `logo-tinta.png` y `codigo-barras.png` a una URL pública estable (S3/CloudFront).
- [ ] Cargar las plantillas en SES y probarlas en Gmail (web y celular), Outlook (escritorio y web) y Apple Mail.
- [ ] Revisar que no caigan en spam (M-224): DMARC/SPF y la parte de texto.
