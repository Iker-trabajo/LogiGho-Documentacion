# Decisiones de Autenticación (ADR)

Decisiones del dominio de cuentas ([`backend/autenticacion.md`](../../../backend/autenticacion.md)), en el formato de la [plantilla ADR](../../../plantillas/PLANTILLA-ADR.md).

| ADR | Decisión |
|---|---|
| [ADR-AUT-001](ADR-AUT-001-plantillas-de-correo-dentro-de-la-lambda.md) | Las plantillas de los correos viven dentro de la lambda (no en SES ni en S3) |
| [ADR-AUT-002](ADR-AUT-002-nombre-fecha-imagenes-y-boton-copiar.md) | Nombre desde Users, fecha en hora de Colombia, imágenes en el sitio de front y un solo botón de copiar |

Otras decisiones del dominio (OTP propio, un solo flujo de recuperación, rol "Red social", roles asignables) están en la spec 008 del workspace y en [`backend/autenticacion.md`](../../../backend/autenticacion.md).
