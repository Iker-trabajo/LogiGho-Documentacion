---
autor: Iker Acevedo
fecha_creacion: 2026-10-22
ultima_actualizacion: 2026-10-22
estado: diseño
---

# Observabilidad — API pública de guías

## Logs

- JSON estructurado en CloudWatch con `traceId`, `clienteId`, `idTienda`, `solicitudId`, `numeroGuia`, `transportadora`.
- Datos personales (nombre, teléfono, dirección, correo) enmascarados.
- Request/response completos de transportadoras en S3 ([ADR-005](adr-005-trazas-transportadoras.md)); en CloudWatch solo resumen.
- Access logs y X-Ray en el stage del API Gateway (hoy apagados en `rl7zt6zak5`).

## Métricas

- Guías creadas, fallidas e inciertas por transportadora y cliente.
- Latencia p50/p95/p99 por transportadora.
- Rechazos por cupo y uso de cupo (%).
- Profundidad de SQS, DLQ y edad del mensaje más viejo.
- Etiquetas pendientes.
- Bloqueos por WAF y por usage plan.

## Alarmas (SNS → Slack y correo)

| Alarma | Umbral inicial |
| --- | --- |
| Tasa de error | > 2% en 5 min |
| Latencia p95 de creación | > 15 s (decisión de Iker 2026-10-25: el POST espera hasta 20 s al worker; con 8 s la alarma saltaba con cualquier guía) |
| DLQ | > 0 mensajes |
| Solicitudes `INCIERTA` | > 0 por más de 10 min |
| Transportadora no disponible | circuito abierto |
| Throttles / errores de Lambda | > 0 |
| Cupo de un cliente | 80% y 100% |

## Dashboard

`ApiPublicaGuias-<env>` en CloudWatch, con las métricas de arriba. El dashboard de cupo para gerencia llega en F4.
