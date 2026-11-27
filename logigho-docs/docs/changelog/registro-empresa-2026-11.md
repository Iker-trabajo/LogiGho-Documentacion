# Registro como persona natural o empresa (nov 2026)

> **Estado (2026-11-27):** backend **desplegado en PreProd** (PR 802 de Lambdas). Front **terminado y validado en localhost**, con su PR a `preprod` pendiente de abrir y desplegar en Vercel. **Aún no está en producción.**

## [2026-11-27] — Registro como persona natural o empresa

### Nuevas funcionalidades
- **Registro como empresa.** `/app/registro` pregunta primero «¿Cómo te registras?» (Persona natural o Empresa). La empresa da razón social, **NIT con dígito de verificación**, representante legal y nombre comercial (opcional), y luego lo de siempre (ciudad, correo, teléfono, cómo se enteró, cuándo empieza).
- **Validación del NIT** con el algoritmo de la DIAN, en el front y en el servidor; se guarda normalizado (`900123456-8`).
- **`Users` gana `tipoPersona`** (`Natural` | `Juridica`) y, para empresa, `tipoDocumento: NIT`, `documento` y el subdocumento `empresa`. `RegisterContact` recibe los datos de empresa al confirmar.

### Cambios
- **Persona natural queda igual**: las 6 preguntas de siempre. Solo se agrega `tipoPersona: "Natural"` al usuario nuevo.
- Los campos nuevos del API son **opcionales**: un cliente que no los envía se trata como persona natural.
- La empresa entra con el rol `Red social` y sin tiendas, como cualquier registro; Comercial le da el acceso.

### Correcciones
- Ninguna.

### Limitaciones conocidas
- **`/registerContact` descarta los campos de empresa** (la lambda es net8 con Native AOT; M-421): una empresa que no termina el registro llega a Comercial solo con el nombre comercial.
- **No se bloquea un NIT repetido** ni se consulta la DIAN o el RUES (M-422).
- Los documentos de la empresa (RUT, cámara de comercio…) **no** se piden en el registro: ya existen en `TiendaDocuments`.

### Orden de salida a producción
1. Backend: `release/2026-11-26-autenticacion` a `main` (CD-Prod con aprobación manual).
2. Front: `preprod` a `main` (merge sin fast-forward). **Nunca el front antes del backend.**

### Documentación
- [API de autenticación: persona natural o empresa](../api/autenticacion.md#persona-natural-o-empresa-spec-039)
- [Backend de autenticación: registro como persona natural o empresa](../backend/autenticacion.md#registro-como-persona-natural-o-empresa-spec-039)
- [Front de autenticación: persona natural o empresa](../frontend/autenticacion.md#persona-natural-o-empresa-spec-039)
- [Pruebas E2E: sección 8](../guias/pruebas-autenticacion-e2e.md#8-registro-como-persona-natural-o-empresa-spec-039)
- [ADR-AUT-004](../arquitectura/adr/autenticacion/ADR-AUT-004-registro-persona-natural-o-empresa.md)
