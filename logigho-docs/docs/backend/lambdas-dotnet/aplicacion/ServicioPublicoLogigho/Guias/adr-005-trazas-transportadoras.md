---
autor: Iker Acevedo
fecha_creacion: 2026-10-22
estado: aceptada
---

# ADR-005 — Trazas HTTP de transportadoras en S3

**Fecha:** 2026-10-22
**Estado:** Aceptada

---

## Contexto

Para soporte se necesita el request y la response exactos enviados a la transportadora. Hoy las lambdas imprimen el body en CloudWatch.

Guardarlo en Mongo no escala: ~50k pedidos/mes × 10–20 KB por traza = ~150k documentos y 2–3 GB en 3 meses, sobre un DocumentDB con 4 GiB de RAM y caché ya al 15%.

## Decisión

- **Traza completa en S3**, JSON comprimido: `trazas/{yyyy}/{MM}/{dd}/{numeroGuia}/{solicitudId}-{paso}.json.gz`. Headers de credenciales enmascarados. Lifecycle de 90 días.
- **Mongo (`ApiSolicitudesGuia`)** guarda solo la key de S3, el código HTTP y la latencia.
- **CloudWatch** recibe una línea estructurada sin body (guía, transportadora, status, ms) para alarmas y búsquedas.
- **Soporte:** `GET /admin/v1/guias/{numero}/trazas` devuelve el request exacto enviado.

## Consecuencias

- Costo de centavos al mes, incluso con 50k pedidos/día.
- La búsqueda por guía pasa por `ApiSolicitudesGuia`, que tiene índice.
