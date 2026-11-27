---

## Autor: Iker Acevedo
Fecha creacion: 2026-11-26
Estado: aceptada

# ADR-AUT-004 — Registro como persona natural o empresa

**Autor:** Iker Acevedo (decisión) · redactado con Claude
**Fecha:** 2026-11-26
**Estado:** Aceptada (backend desplegado en PreProd el 2026-11-27; front en PR a `preprod`; producción pendiente)

---

## Contexto

El registro por autoservicio (spec 008) solo servía a una persona: no había dónde dar la razón social ni el NIT de una empresa, y Comercial tenía que pedirlos por otro medio. Se pidió permitir el registro **como persona natural o como empresa**, dejando el registro actual **tal cual** y la empresa como un **adicional**. Los documentos de la empresa (RUT, cámara de comercio…) ya existen en la colección `TiendaDocuments` y la validación por Comercial, las tiendas y la invitación de usuarios se definen después: **no entran aquí**.

## Opciones consideradas

### Opción A — Campo `tipoPersona` y subdocumento `empresa` en `Users`, campos opcionales (elegida)

`Users` gana `tipoPersona` (`Natural` | `Juridica`; ausente = `Natural`) y, en empresa, `tipoDocumento: "NIT"`, `documento` (el NIT) y `empresa { razonSocial, nombreComercial?, representanteLegal { nombre } }`. Todos los campos del contrato son opcionales.

**Pros:** sin migración; los usuarios y clientes anteriores no cambian; reutiliza `tipoDocumento` y `documento`, que ya existen (hay usuarios con `NIT`); el subdocumento deja la empresa agrupada.
**Contras:** `Users` no tiene índice por `documento`, así que un NIT repetido no se detecta (M-422).

### Opción B — Colección aparte para empresas

**Pros:** `Users` queda intacto.
**Contras:** otra colección, otro repositorio y otro cruce que mantener antes de saber cómo se relacionan empresa, tiendas y usuarios (decisión pendiente). Descartada por ahora.

### Opción C — Reutilizar el campo `tipo` de `Users`

**Contras:** `tipo` ya existe con valores sueltos (`["Tienda"]`, listas de roles, nulos); mezclar significados es un riesgo. Descartada.

## Decisión

Opción A.

- **Registro público único** (`/app/registro`), con una primera pregunta que elige el tipo; no hay un formulario aparte.
- **NIT validado con el dígito de verificación de la DIAN**, en el front y en el servidor, y guardado normalizado (`NNNNNNNNN-D`). No se consulta la DIAN ni el RUES.
- **La empresa entra como `Red social`, sin tiendas**, igual que una persona natural: Comercial le da el acceso real.
- **El backend se despliega antes que el front.** Un backend anterior descartaría en silencio los campos de empresa.
- **`/registerContact` no se toca** (M-421): es net8 con Native AOT, y la regla del repo obliga a subir a net10 toda lambda que se modifique; publicarla con Docker es otra tarea. Consecuencia: una empresa que no termina el registro llega a Comercial solo con el nombre comercial; si lo termina, la fila se completa al confirmar.
- El tipo desconocido y la empresa incompleta son `400 DATOS_INVALIDOS`; la respuesta sigue siendo `202` igual exista o no el correo.

## Consecuencias

- Los lectores de `Users` deben tratar la ausencia de `tipoPersona` como `Natural`.
- Mejoras abiertas: M-421 (`/registerContact` con los campos de empresa), M-422 (NIT repetido) y M-423 (`ApiLambdaTokenCognitoAOT.Tests` no compila: plantilla de AWS).
- Los logs no llevan NIT, razón social ni representante (probado).

## Verificación

Backend: 357 + 41 + 32 pruebas en verde (Debug y Release) y `ApiLambdaRegistro` publicado a PreProd el 2026-11-27. Front: 80 pruebas del registro, build de producción y recorrido con Playwright. Pendiente: las pruebas 48 a 58 de la [guía E2E](../../../guias/pruebas-autenticacion-e2e.md#8-registro-como-persona-natural-o-empresa-spec-039).
