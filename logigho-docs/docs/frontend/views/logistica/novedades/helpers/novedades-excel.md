## Autor: Yadilton Torres

Fecha creacion: 2026-10-30

Estado: desarrollo

## Helper: novedades-excel.helper.ts

**Ubicación:** `helpers/novedades-excel.helper.ts`
**Tests:** `novedades-excel.helper.spec.ts`

---

## ¿Qué hace?

Descarga en Excel **solo lo que se ve** con la pestaña y los filtros actuales (todas sus páginas), nunca la base completa. El botón del encabezado muestra cuántas guías va a descargar ("Exportar 624 guías").

La librería `xlsx` se carga con `import()` dinámico solo al exportar, para no pesar en la carga de la vista.

---

## Columnas

Guía · Transportadora · Tienda · Motivo · Motivo completo · Estado transportadora · Ciudad · Fecha novedad · Días abierta · Intentos de entrega · Seguimiento · Prioridad · Consultas · Consulta fallida · Guía PDF

**No incluye** datos del destinatario (nombre, teléfono, dirección) ni fotos: el archivo sale de la plataforma y se comparte.

---

## Funciones

| Función | Qué hace |
| ------- | -------- |
| `filasExcel(filas, ahora)` | Arma una fila por guía con las columnas anteriores |
| `fechaExcel(iso)` | Fecha en hora Colombia, formato `dd/mm/aaaa HH:mm` |
| `descargarExcel(filas, ahora)` | Genera y descarga el archivo (hoja "Novedades", anchos de columna fijos) |
