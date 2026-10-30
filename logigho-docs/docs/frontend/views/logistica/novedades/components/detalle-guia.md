## Autor: Yadilton Torres

Fecha creacion: 2026-10-30

Estado: desarrollo

## Panel: detalle de la guía

**Ubicación:** dentro de `novedades.component.html` (panel lateral; a pantalla completa en celular)

---

## ¿Qué hace?

Muestra todo lo que se sabe de una guía. **Cambia según la transportadora**: cada una muestra solo lo que de verdad trae, sin tarjetas vacías con "—".

| | Interrapidísimo | Envía | Servientrega |
| - | --------------- | ----- | ------------ |
| Tarjetas de color | Abierta · Consultas · Prioridad · Seguimiento | Abierta · Intentos · Etapa "Novedad" | Desde el aviso · Intentos · Etapa "Regresa" |
| Aviso de arriba | — (el motivo va en el chip) | Mensaje de la transportadora | "Este envío viene de regreso: <motivo>" |
| Envío | Tienda, estado, tipo de entrega, mensajero, intentos | Igual, sin mensajero | Igual, sin mensajero |
| Mapa | Sí, por intento | No (Envía no manda ubicación) | No |
| Fotos de la novedad | Sí (S3) | No | No |
| Trazabilidad | Sí | No | No |
| PDF y destinatario | Sí | Sí | Sí |

---

## Intentos de entrega con mapa (Inter)

- Por defecto el mapa muestra el **último intento** (`UltimoMotivo.Geo`).
- Si hay más de un intento, debajo sale la lista del más reciente al más viejo (motivo, fecha, ciudad). **Tocar uno cambia el mismo mapa** a esa ubicación, junto con el enlace "Abrir en Maps".
- Los intentos sin coordenadas aparecen en gris con "Sin ubicación" y no se pueden tocar.
- Al pasar a otra guía con ▲▼ vuelve al último intento.

El mapa es un `iframe` de Google Maps; la URL segura se guarda por ubicación para que el iframe no se recargue en cada ciclo.

---

## Fotos de la novedad (Inter)

- Vienen de `UltimaRespuesta.gestion_app[]`: cada una trae `s3_key`, fecha, tipo y descripción.
- Al abrir el detalle, cada foto se pide a S3 (`wsinter-evidencias-prod`) con `getObject`. Mientras carga dice "Cargando foto…"; si falla, "No se pudo cargar la foto" y el motivo queda en la consola.
- Tocar una miniatura la abre en grande. Se cierra con ✕, tocando fuera o con Esc; **Esc cierra primero la foto y después el panel**.
- Las fotos se guardan solo en memoria mientras se está en la vista: no van al Excel ni al navegador.

---

## Acciones

- Copiar guía, teléfono o dirección (con ✓ unos segundos).
- WhatsApp al destinatario (`numeroWhatsapp`).
- Ver la guía en PDF.
- Navegar con ▲▼ (también con el teclado) sin cerrar el panel; la tabla avanza sola a la página de la guía.
- Mientras el panel está abierto, la página de atrás no se desplaza.
