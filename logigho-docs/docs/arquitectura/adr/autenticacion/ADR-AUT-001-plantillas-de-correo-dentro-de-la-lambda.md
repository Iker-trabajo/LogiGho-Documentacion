---

## Autor: Iker Acevedo
Fecha creacion: 2026-11-09
Estado: aceptada

# ADR-AUT-001 — Las plantillas de los correos viven dentro de la lambda

**Autor:** Iker Acevedo (decisión) · redactado con Claude
**Fecha:** 2026-11-09
**Estado:** Aceptada

---

## Contexto

El equipo de front diseñó los cuatro correos de seguridad (código de registro, cuenta existente, código de recuperación y aviso de contraseña cambiada) como HTML y texto plano con variables `{{nombre}}`, `{{codigo}}`, etc. Había que decidir **dónde vive ese HTML y quién lo rellena**. Es una decisión de seguridad, no solo de diseño: estos correos llevan el código que da acceso a una cuenta.

## Opciones consideradas

### Opción A — Archivos dentro de la lambda (elegida)

Los ocho archivos (`01`–`04`, `.html` y `.txt`) están en `Autenticacion.Nucleo/Plantillas/`, viajan embebidos en la dll y el código los rellena.

**Pros:**
- El diseño y el código que lo rellena se versionan, prueban, despliegan y **revierten juntos**: no puede haber una plantilla que pida una variable que el código ya no entrega.
- Se prueba con `dotnet test` (312 pruebas: variables sin reemplazar, codificación del HTML, asuntos, fechas).
- No hay infraestructura nueva ni permisos nuevos (ni `ses:CreateTemplate`, ni lectura de S3).
- El cambio de un correo de seguridad pasa por un PR con revisión.

**Contras:**
- Cambiar el diseño exige un PR en el repo de Lambdas y un despliegue; front no puede publicarlo solo. Se atenúa porque el cambio es reemplazar archivos, sin tocar código.

### Opción B — Plantillas de SES (`CreateTemplate` y `SendTemplatedEmail`)

Se crean en AWS y el código solo manda las variables.

**Pros:** cambiar el diseño no requiere desplegar la lambda; es el formato que contempla el LEEME de front.
**Contras:** hay que crearlas y mantenerlas en **cada cuenta** (PreProd y producción), por Terraform o a mano; el diseño queda fuera de git y de las pruebas; si el código y la plantilla se desincronizan, el envío falla en producción; la reversión son dos pasos distintos.

### Opción C — HTML en S3 o en el sitio de front, descargado al enviar

**Pros:** front controla el diseño.
**Contras:** una dependencia de red más en cada envío (si falla, no sale el correo de seguridad); más difícil de probar y revertir; superficie nueva de ataque (quien edite el archivo cambia el correo).

## Decisión

**Opción A.** La plantilla es parte del código que protege el acceso a la cuenta; debe tener la misma revisión, pruebas y reversión que el código. Front conserva la autoría del diseño y lo entrega por PR (reemplazar los 8 archivos).

## Cómo funciona

- `PlantillaCorreo` rellena `{{variable}}`. En el HTML todo valor se **codifica** (un nombre con `<script>` no rompe el correo); en el texto va tal cual.
- Una variable que la plantilla pide y el código no entrega lanza un error: preferimos no enviar a enviar un correo con `{{codigo}}` a la vista.
- No se usan plantillas de SES, aunque usamos la misma forma `{{variable}}`: si algún día se pasa a la opción B, las plantillas sirven sin cambios.

## Consecuencias

- Front entrega el diseño por PR al repo de Lambdas. Si pesa, se reevalúa la opción B como historia aparte (costo: Terraform de plantillas en dos cuentas y un paso de despliegue más).
- Cada cambio de diseño es un despliegue de la lambda de registro (rutina: CD de autenticación).
- La ruta vieja `/cambio-pass` conserva su texto anterior hasta que se retire (M-232).
