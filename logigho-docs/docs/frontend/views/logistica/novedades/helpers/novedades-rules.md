## Autor: Yadilton Torres

Fecha creacion: 2026-10-30

Estado: desarrollo

## Helper: novedades.rules.ts

**Ubicación:** `helpers/novedades.rules.ts`
**Tests:** `novedades.rules.spec.ts` — funciones puras, sin Angular ni HTTP

---

## ¿Qué hace?

Reglas puras de la vista: misma entrada, misma salida. No llaman al API ni tocan el DOM; el componente y el mapper las usan.

---

## Motivo

| Función | Qué hace |
| ------- | -------- |
| `tieneNovedad(f)` | `true` si trae motivo (Inter) o mensaje de la transportadora |
| `motivoCorto(motivo)` | `"OTROS / RESIDENTE AUSENTE (Agencia)"` → `"RESIDENTE AUSENTE (Agencia)"` |
| `etiqueta(f)` | Texto corto para chips, barras y filtro: Inter → parte después de "/"; Envía → lo que sigue a "novedad:" hasta la coma; Servientrega → lo que sigue a "Devolucion:" |
| `esMensajeDevolucion(texto)`, `enDevolucion(f)` | Detectan el mensaje `Devolucion: <motivo>` y la etapa de devolución |
| `tonoTexto(texto)`, `tono(f)` | Familia del motivo para el color: `rehusado`, `ausente`, `direccion`, `otro` (incluye los motivos de Servientrega: "SE NEGO A RECIBIR", "NO LO CONOCEN", "NO HAY QUIEN RECIBA"…) |

---

## Antigüedad

| Función | Qué hace |
| ------- | -------- |
| `diasAbiertos(f, ahora)` | Días completos desde la novedad (o desde la creación si no hay fecha) |
| `semaforo(dias)` | `≤ 1` día → `ok` (verde), `≤ 3` → `alerta` (amarillo), más → `critico` (rojo) |

---

## Búsqueda de guías

| Función | Qué hace |
| ------- | -------- |
| `guiasBuscadas(texto)` | Separa por coma, punto y coma, espacios o saltos de línea; sin repetidas |
| `unirGuias(texto)` | Una columna pegada de Excel queda en una línea separada por comas |

---

## Fechas, ciudades y contacto

| Función | Qué hace |
| ------- | -------- |
| `fechaIso(valor)` | Normaliza `Date`, `{ $date }`, ISO y "AAAA-MM-DD HH:mm:ss" de `PedidosInter` (hora Colombia) a ISO |
| `diaColombia(iso)` | Día `AAAA-MM-DD` en hora Colombia (UTC-5, sin horario de verano) para el filtro de fecha |
| `ciudadDeEstado(estado)`, `estadoSinCiudad(estado)` | "EN NOVEDAD POR MEDELLIN" → "MEDELLIN" / "EN NOVEDAD" |
| `limpiarCiudad(valor)` | "BUENAVISTA BOYACA\BOYA\COL" → "BUENAVISTA BOYACA" |
| `urlUbicacion(geo)`, `coordenadasDeUrl(url)` | Enlace de Google Maps desde `Geo` y sus coordenadas para el mapa embebido |
| `numeroWhatsapp(telefono)` | Número con indicativo 57 para el enlace de WhatsApp |
| `normalizarTransportadora`, `transportadoraDe` | Nombre de la base → clave y datos de la transportadora |
| `capitalizar`, `iniciales` | Textos para mostrar |
