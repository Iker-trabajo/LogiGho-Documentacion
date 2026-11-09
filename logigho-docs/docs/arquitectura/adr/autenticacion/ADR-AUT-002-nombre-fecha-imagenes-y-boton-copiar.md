---

## Autor: Iker Acevedo
Fecha creacion: 2026-11-09
Estado: aceptada

# ADR-AUT-002 — Nombre, fecha, imágenes y botón de copiar de los correos

**Autor:** Iker Acevedo (decisiones) · redactado con Claude
**Fecha:** 2026-11-09
**Estado:** Aceptada

---

Cuatro decisiones pequeñas que salen de pasar los correos al diseño de front ([ADR-AUT-001](ADR-AUT-001-plantillas-de-correo-dentro-de-la-lambda.md)). Cada una dice qué se eligió, qué se descartó y por qué.

## 1. El nombre sale de Users y viaja en la solicitud

**Contexto.** Las plantillas de recuperación y de aviso saludan por el nombre ("Ana, volvamos a ponerte en ruta"). En el registro el nombre llega en el formulario; en la recuperación solo se conoce el correo.

**Elegido.** Al pedir el código se lee el `username` de la cuenta en `Users` y se guarda en la solicitud (`RegistroSolicitudes`). El aviso de cambio lo toma de ahí, sin otra consulta. Sin nombre se usa "Cliente LogiGho".

**Descartado.** *Quitar el nombre de esas frases:* el LEEME de front lo permite, pero los correos quedan menos cercanos y distintos al registro. *Consultar Users otra vez al avisar:* una lectura más y otro punto de fallo después de que la contraseña ya cambió.

**Por qué es seguro.** El nombre solo se busca cuando la cuenta existe, así que el correo sigue sin revelar quién está registrado (si no existe, no se envía nada y la respuesta es la misma). El nombre es lo que la propia persona escribió al registrarse y se codifica en el HTML.

## 2. La fecha del aviso va en hora de Colombia, calculada a mano

**Contexto.** La guía muestra cuándo cambió la contraseña (`5 nov 2026 · 1:06 p. m.`). La lambda corre en Linux con la cultura del servidor sin garantía.

**Elegido.** UTC-5 fijo (Colombia no tiene horario de verano), con meses y "a. m./p. m." escritos en el código.

**Descartado.** *Zona horaria del sistema (`TimeZoneInfo`):* depende de datos de zona horaria que el runtime puede no traer, y una diferencia de formato cambiaría el correo sin avisar. *UTC:* confunde a quien lee ("la 1 p. m. de Colombia" es la hora que recuerda).

Está cubierto con pruebas (cambio de día, medianoche, mediodía).

## 3. Las imágenes viajan dentro del correo (cid), sin hosting

**Contexto.** El correo necesita tres imágenes: `logo-tinta.png`, `codigo-barras.png` e `icono-copiar.png`. La primera versión las leía de una carpeta del sitio de front (`www.logigho.com/assets/images/correos/`); front no las había publicado (la dirección devolvía la página del front) y los correos de PreProd salieron con imágenes rotas.

**Elegido (2026-11-09, Iker pidió resolverlo sin depender de nadie).** Las imágenes van **dentro del mensaje**: el HTML las referencia como `cid:logo-tinta` y el mensaje (`multipart/related`) las lleva adjuntas con su `Content-ID`. Se guardan embebidas en la dll junto a las plantillas y se envían por SES como mensaje crudo (`Content.Raw`). Pesan ~10 KB en total.

**Por qué.** No dependen de un despliegue de front, de un bucket público ni de que el cliente de correo descargue nada de internet; el correo se ve igual en PreProd y producción desde el primer día. Es el método estándar de los correos con diseño.

**Descartado.** *Carpeta en el sitio de front:* dependencia de otro equipo y de su ciclo de despliegue; imágenes rotas si cambia o borra la ruta. *Bucket público en AWS (S3/CloudFront):* infraestructura nueva y un recurso público más que vigilar para tres PNG fijos. *Base64 dentro del HTML (`data:`):* Gmail y Outlook lo bloquean.

**Contras aceptados.** Algunos clientes muestran las imágenes en línea como adjuntos o piden "mostrar imágenes" (Outlook); cambiar una imagen exige desplegar la lambda. Si algún día se quiere cambiar a direcciones públicas, basta con volver a poner URLs en las variables `logo_tinta_url`, `img_barras_url` e `icono_copiar_url`.

## 4. Un solo botón de copiar, de adorno

**Contexto.** El primer diseño tenía una celda naranja "COPIAR" que repetía el código debajo de las casillas. Iker la encontró recargada.

**Elegido.** Un botón pequeño con el icono de copiar junto a las casillas. **Un correo no puede copiar con un clic** (no admite programación): el icono indica la acción y el código se copia seleccionando las casillas o con la sugerencia de copiar que ofrece Gmail en Android.

**Descartado.** *Mantener la celda con el código repetido:* ruido visual y dos veces el código en pantalla. *Un enlace que "copie":* los enlaces de un correo de seguridad deben ser solo el del flujo; no se agregan otros.
