---

## Autor: Yadilton Torres

Fecha creacion: 2026-11-11  
Estado: activo  
Tipo: componente

# Componente: Accesos en LOGI (formulario)

**Selector:** `app-accesos-logi`  
**Ubicación:** `src/app/shared/components/accesos-logi/`  
**Dónde se usa:** [Default Layout](../layout/default-layout.md) (al iniciar sesión) y el menú de la cuenta del [Default Header](../layout/default-header.md)

---

## ¿Qué hace?

Es el formulario con el que una persona nueva entrega los datos para activar su tienda. Reemplaza el Google Forms "Registro de Tiendas". Sale lleno con lo que ya hay en `RegisterContact` y `Users`, y guarda en `RegisterContact`.

Cuando alguien lo completa por primera vez, a los roles **Comercial** y **COO** les llega una notificación para que le respondan (ver [Notificaciones](#notificacion-a-comercial-y-coo)).

---

## Cuándo sale

Lo decide `revisarAccesos()` en el Default Layout, después de cargar el usuario:

| Condición | Resultado |
| --- | --- |
| La cuenta tiene **un solo rol y es `Red social`** y su `RegisterContact` **no** tiene `fechaAccesos` | Sale al entrar, **obligatorio**, en cada inicio de sesión y en cada recarga hasta que lo complete |
| La consulta a `RegisterContact` falla | Sale igual (el formulario va antes que la guía) |
| Ya lo completó (`fechaAccesos` existe) | No sale; pasa a la guía |
| Cualquier otro rol (o Red social junto con otro rol) | No sale ni el formulario ni la guía |

**Orden:** primero el formulario y, cuando lo termina, la guía de bienvenida de Red social (`app-bienvenida-red-social`). La guía nunca sale encima del formulario.

**Desde el menú de la cuenta** ("Accesos en LOGI", entre Mi Perfil y Wallet) se puede abrir cuando se quiera. Para quien tiene solo Red social y aún no lo completó en la sesión, sigue siendo obligatorio; para los demás se cierra normal.

### Obligatorio

Con `obligatorio = true` la ✕ se ve, pero no cierra: muestra una franja naranja *"La primera vez es obligatorio llenar este formulario para activar tu tienda. Solo toma un par de minutos."* con el botón "Entendido". Esc y clic afuera tampoco cierran. Se cierra con **"Entrar a LOGI"** después de guardar.

---

## Entradas y salidas

| Nombre | Tipo | Descripción |
| --- | --- | --- |
| `@Input() correo` | `string` | Correo de la sesión; con él se busca su `RegisterContact` |
| `@Input() usuario` | `UsuarioAccesos` | Datos de `Users`: nombre, teléfono, documento, `autorizaDatos` y `tieneTienda` |
| `@Input() obligatorio` | `boolean` | No se puede cerrar hasta guardar |
| `@Output() cerrar` | `void` | Se cerró (o terminó) |

`tieneTienda` es verdadero si la cuenta tiene alguna tienda real ("Todas", "Ninguna" y "0" no cuentan).

---

## Pasos

| Paso | Campos |
| --- | --- |
| 1. Tu tienda | Nombre de la tienda, tipo de negocio, ciudad desde donde despacha, dirección de recolección |
| 2. Tus envíos | ¿Requiere fulfillment?, paquetes al mes |
| 3. Tu usuario | Nombre completo, cédula o NIT, teléfono / WhatsApp, correo (de la cuenta, solo lectura), autorización de datos |
| 4. Confirmar | Resumen por grupo; tocar un grupo lleva a ese paso. Botón **Enviar** |

### Todo es obligatorio

"Siguiente" no avanza si falta algo; cada campo con problema se pone en rojo con su mensaje y abajo dice *"Completa los campos marcados para continuar"*. Tampoco se puede saltar a un paso si los anteriores no están completos.

| Campo | Regla |
| --- | --- |
| Nombre de la tienda | Mínimo 2 caracteres |
| Tipo de negocio | Elegir uno |
| Ciudad | Elegir una |
| Dirección | Mínimo 6 caracteres y con número (calle y número) |
| Fulfillment | Sí o No |
| Paquetes al mes | Elegir uno |
| Nombre | Solo letras y espacios, 3 a 80 |
| Cédula o NIT | 5 a 12 números |
| Teléfono | 10 números |
| Autorización de datos | Marcada |

Al salir de un campo y al guardar se quitan los espacios del inicio, del final y los repetidos.

### Ciudad

Un solo campo con lista desplegable:

- Al hacer clic muestra **Ciudades principales** (Bogotá, Medellín, Cali, Barranquilla, Bucaramanga, Cartagena, Pereira, Manizales, Cúcuta) con su departamento.
- Al escribir busca entre todas las de la colección `Ciudades` (sin tildes) y muestra hasta 8 resultados, primero los que empiezan igual.
- Flechas para moverse, Enter elige, Esc cierra la lista sin cerrar el formulario; clic afuera también la cierra.
- La ciudad elegida queda como un dato lleno con "Cambiar".
- Los nombres se muestran limpios: `SANTIAGO DE CALI` → **Cali**, `BOGOTA` → **Bogotá**, el resto en formato normal. Así no salen repetidas.

### Datos que ya teníamos

Lo que vino lleno lleva la etiqueta **"✓ Guardado"**; si la persona lo cambia, **"Se actualizará"**. El texto del paso dice que algunos datos vienen llenos y se pueden cambiar.

- Cédula: de `Users.documento` si `RegisterContact` no la tiene.
- Habeas data: marcado si `Users.autorizaDatos` es `true`.
- **Nombre de la tienda bloqueado** si la persona ya tiene tienda en LOGI y ese nombre ya estaba registrado.

---

## Guardado

`AccesosLogiRepository.guardar(original, datos)`:

- Si la persona **no tiene** `RegisterContact`, se crea uno con todo y estado `Registrado en plataforma`.
- Si ya tiene, se envía solo lo que cambió respecto a lo que había en `RegisterContact`. También se guarda lo que vino de `Users` aunque no lo cambie.
- Un campo que la persona dejó vacío no borra un dato que ya existía.
- Siempre se guarda `fechaAccesos` (fecha ISO) para marcarlo como completado.
- `GUARDADO_ACTIVO` apaga la escritura: se muestra el resumen pero no se guarda.

Campos: ver [Campos de RegisterContact](../views/comercial/panel-comercial/registercontact-campos.md).

### Notificación a Comercial y COO

La **primera vez** que alguien lo completa (su registro no tenía `fechaAccesos`) se envía una notificación `NUEVO_REGISTRO_ACCESOS` a los roles `Comercial` y `COO` con: tienda, nombre, correo, teléfono, ciudad, tipo de negocio y paquetes al mes. Quien la responda le envía a la persona un `REGISTRO_RESPONDIDO` y se cierra. Si el aviso falla, el formulario ya quedó guardado y no se le muestra error a la persona. Ver [Servicio de Notificaciones](../core/notificaciones-service.md).

---

## Endpoints

| Acción | Llamada |
| --- | --- |
| Buscar su registro | `GET metodoGenerico?coleccion=RegisterContact&email=<correo>` |
| ¿Ya lo completó? | La misma consulta; mira `fechaAccesos`. 404 = no tiene registro |
| Ciudades | `consultarGenericoCacheado('1', 'metodoGenerico?coleccion=Ciudades')` |
| Guardar | `actualizarGenerico` en `metodoGenerico?coleccion=RegisterContact`, o `insertarGenericoST` en `registerContact` si no existe |

---

## Diseño

Fondo blanco (igual que el resto de la página), pasos arriba con barra de avance, tarjetas por pregunta, esquinas de 8 px, letra 500/600/700. Pantalla final con confeti. En celular sale desde abajo.

---

## Historial de cambios

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-11-11 | Yadilton Torres | Formulario Accesos en LOGI en 4 pasos, lleno con RegisterContact y Users |
| 2026-11-11 | Yadilton Torres | Sale solo a quien tiene únicamente el rol Red social y no lo ha completado; obligatorio; antes de la guía |
| 2026-11-11 | Yadilton Torres | Todos los campos obligatorios; ciudad con buscador desplegable; fondo blanco; etiqueta "✓ Guardado" |
| 2026-11-11 | Yadilton Torres | Notificación a Comercial y COO al completarlo por primera vez |

---

## Observaciones

- **Correos con mayúsculas en la base:** la búsqueda manda el correo en minúsculas; un registro guardado con mayúsculas puede no encontrarse y se crearía otro.
- **La regla de "un solo rol"** depende de qué roles recibe una cuenta nueva al registrarse. Si se le asigna otro rol automático, el formulario no sale.
- **Backend debe aceptar `fechaAccesos`** en `RegisterContact`; si no lo guarda, el formulario sale siempre.
