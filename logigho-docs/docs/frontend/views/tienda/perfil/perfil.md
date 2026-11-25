---

## Autor: Yadilton Torres

Fecha creacion: 2026-11-19  
Estado: activo  
Tipo: vista

# Vista: Mi perfil

**Selector:** `app-perfil`  
**Ubicación:** `src/app/views/tienda/perfil/`  
**Ruta:** `/app/tienda/perfil`  
**Acceso:** cualquier usuario con sesión; ve solo las tiendas de su cuenta (`tiendas_asignadas`)

---

## ¿Qué hace?

Es la página donde la persona deja lista su cuenta para vender en LOGI. En un solo lugar:

- Completa **sus datos** (nombre, documento, teléfono, foto y banner).
- **Crea su tienda** y edita teléfono, ciudad y dirección mientras no esté activa.
- **Sube los documentos** de la tienda y de las personas autorizadas.
- **Agrega las cuentas bancarias** donde recibe el dinero.

Arriba muestra el avance en una rueda de porcentaje y el siguiente paso que le falta.

---

## Archivos

Organizado como los demás módulos del proyecto (pantalla en la raíz, y `components/`, `helpers/`, `models/`, `repository/` y `styles/`):

```
tienda/perfil/
  perfil.component.ts / .html / .spec.ts
  perfil.validaciones.spec.ts
  components/detalle-tienda/      (+ .spec.ts)
  helpers/perfil.rules.ts         (+ .spec.ts)
  models/perfil.models.ts
  repository/perfil.repository.ts (+ .spec.ts)
  styles/perfil.scss, perfil.mobile.scss
```

| Archivo | Responsabilidad |
| --- | --- |
| `perfil.component.ts` / `.html` | Pantalla: avance, tiendas, documentos, cuentas y los modales para crear o editar |
| `helpers/perfil.rules.ts` | Reglas puras: documento por tipo, remitente, detalle de la tienda, estado de los 4 documentos y resumen por tienda, filtros, paginación y constantes (`MAX_ARCHIVO_MB`, `BANCOS_BILLETERA`, `REGLAS`) |
| `models/perfil.models.ts` | `TiendaPerfil`, `CuentaPerfil`, `DocumentoPerfil`, `PropietarioPerfil`, `PasoPerfil`, `ModalPerfil`, `ResumenDocumentosTienda` |
| `repository/perfil.repository.ts` | Escrituras: crear y editar tienda, mis datos, fotos (y volver a la predeterminada), documentos (S3 + `TiendaDocuments`) y cuentas |
| `components/detalle-tienda/` | "Detalle y documentos" de una tienda: pestañas Datos, Documentos tienda, una por autorizado y "+ Autorizado" |
| `styles/perfil.scss` / `perfil.mobile.scss` | Estilos de escritorio y celular (≤768 px) |

### Pruebas (61)

| Archivo | Qué prueba |
| --- | --- |
| `perfil.component.spec.ts` | Que la pantalla carga, Cambiar contraseña y foto/banner predeterminados |
| `perfil.validaciones.spec.ts` | Modales con revisión en vivo y cuentas inactivas |
| `helpers/perfil.rules.spec.ts` | Documento por tipo, remitente, detalle de tienda, filtros, estado y resumen de documentos |
| `repository/perfil.repository.spec.ts` | Crear tienda (forma, orden e Id), nombre repetido, editar tienda, documentos y cuentas (sin Id no deja) |
| `components/detalle-tienda/detalle-tienda.component.spec.ts` | Tarjetas y avance, autorizados, subida y sus errores |

Componentes compartidos que usa: [Lista de opciones](../../../components/lista-opciones.md), `app-confirmar-cambios`, `app-paginador`, `app-direccion-estandarizada` (Armar dirección del 1 a 1), `app-cambio-contrasena` y la directiva `appModalTeclado`. Para crear la tienda usa la [Forma de la tienda](../../../core/forma-tienda.md).

---

## Pantalla

### Avance

- **Rueda de porcentaje:** cuenta los 5 pasos (mis datos, crear tienda, foto y banner, documentos y cuenta bancaria). Va en naranja mientras falta algo y pasa a verde al 100 %. El mismo avance se publica en el encabezado (`AvancePerfilService`).
- **Siguiente paso:** tarjeta azul con el paso que falta y su botón.
- **Pasos:** el actual en azul con pulso, los completos en verde y los pendientes en gris.

!!! note "Colores"
    Desde el 24 de noviembre de 2026 Mi perfil usa los colores del [rediseño «La Red»](../../../tema-la-red.md) (variables `--lg-*`), así se ve bien de noche y de día. No quedan colores fijos (`#fff`, `--logigho-*`): fondos con `--lg-superficie*`, "en revisión" con `--lg-alerta-*`, validado con `--lg-exito-*`, faltante con `--lg-error-*` y enlaces o etiquetas con `--lg-info-*`.

### Foto y banner

- Se cambian en **Editar mis datos**. Se guardan en `PerfilesUsuario` (`avatarUrl`, `bannerUrl`), la misma colección del espacio social, así la foto es una sola en toda la plataforma (`FotoPerfilService`).
- **Usar la predeterminada:** debajo de «Foto de perfil» y de «Banner». Si la persona tiene una imagen propia, sale el botón **«Usar la predeterminada»**; al tocarlo queda marcado (*«Se usará la predeterminada · deshacer»*) y, al guardar, se **vacía solo esa URL** (`FotoPerfilService.quitar`). Vuelven la inicial y el banner de siempre; el archivo subido no se borra.
- Si no tiene imagen propia, sale *«✓ Ya usas la predeterminada»* (solo informa).
- Elegir otra imagen deja sin efecto lo marcado. En el resumen antes de guardar sale *«Foto de perfil: La actual → Predeterminada»*.
- Medidas recomendadas: foto **500 × 500 px** (se ve en círculo de 112 px, 84 en celular) y banner **1920 × 400 px** (160 px de alto, 120 en tableta, 112 en celular; lo importante, en el centro). PNG, JPG o WebP, máximo 5 MB.

### Tiendas

- **Crear tienda:** nombre, correo (fijo, el de la cuenta), tipo de negocio (varios y "Otro: cuál"), teléfono, ciudad y dirección. Ciudad y dirección son obligatorias.
- La tienda **nace inactiva** y asignada al usuario. Comercial la activa.
- **Id:** lo calcula el front como el más alto más 1, igual que Administración → Tiendas. Si una tienda vieja no tiene Id, Seguimiento se lo pone al guardarla.
- **Forma y orden:** nace con los 17 campos y en el mismo orden que en Administración → Tiendas (`armarTiendaNueva`). Ver [Forma de la tienda](../../../core/forma-tienda.md).
- **Nombre único** en toda la plataforma, sin importar mayúsculas, tildes ni espacios.
- **`TipoTienda`:** se guarda como en producción (`Propia`, `Dropshipping`, `Proveedor` u otro).
- **Teléfono:** el ejemplo dice «Ej: 300 123 4567» y los textos de ejemplo van en gris claro, para que no parezcan un dato ya escrito.
- **Tienda activa:** no se edita; sale "comunícate con un asesor".

### Documentos

- Resumen con una tarjeta por tienda (3 por página) y una etiqueta por documento: validado (verde), en revisión (ámbar) o faltante (rojo).
- **Detalle y documentos:** tarjetas con Ver, Subir o Reemplazar; aviso "todo a nombre del representante legal"; descarga de todo en un `.zip` con carpetas.
- Se guarda la ruta del archivo en `Documento` (compatible con Documentos Tiendas). No deja subir si la tienda no tiene Id.

### Cuentas bancarias

- Mismo formato del formulario de siempre (`creacioncuentabancaria`).
- **Tienda sin Id:** bloqueada.
- **Repetida:** no se registra la misma cuenta en el mismo banco y la misma tienda. Un mismo celular sí puede estar en Nequi y en Daviplata.
- **Nequi y Daviplata** pasan solos a "Billetera Virtual" (el número es el celular).
- **Titular** en MAYÚSCULAS.
- **Cuentas viejas incompletas:** muestran "Datos incompletos".
- **Cuentas inactivas:** **no se muestran**. Si la persona intenta registrar otra vez una cuenta inactiva, sale *"Esta cuenta ya existe en esa tienda pero está inactiva. Comunícate con un asesor para activarla."* Ver [Estado de las cuentas bancarias](cuentas-estado.md).
- 2 cuentas por página.

---

## Modales

Crear o editar tienda, editar mis datos, subir documento y agregar cuenta.

### Revisión en vivo

| Comportamiento | Detalle |
| --- | --- |
| Al guardar | Se validan todos los campos; los que fallan quedan en rojo y se lleva al primero (`irAlPrimerError`) |
| Mientras escribe | Solo se revisan de nuevo **los campos que ya están en rojo** (`revisarEnVivo`). Si quedaron bien, se quita el rojo; si el problema cambió, cambia el mensaje |
| Campo corregido | Borde verde y *"✓ Listo"* por 2,5 s (`corregidos`) |
| Campos sin tocar | No se marcan en rojo antes de tiempo |
| Resumen | Encima de los botones: *"Faltan corregir N campos · Ver"* (lleva al primero) o *"Todo listo, ya puedes guardar."* |
| Archivos | Sus errores se quitan al elegir otro archivo, no al escribir |
| Nombre rechazado | Si el backend dice que el nombre ya existe (`NombreTiendaRepetidoError`), sigue en rojo mientras no se cambie (`nombreRechazado`) |

La revisión se dispara desde el contenedor del modal con `(input)`, `(change)`, `(click)` y `(keyup.enter)`, así cubre campos de texto, listas y chips sin tocar cada campo.

### Listas

Todas las listas de los modales usan `[siempreAbajo]="true"`: abren hacia abajo con su alto normal y **no mueven el scroll**; si no cabe, la persona baja. La de ciudades (más de mil) se pinta por tandas. En celular salen como hoja inferior.

### Validaciones comunes

- Teléfono: 10 números, empieza por 3 (celular) o 60 (fijo).
- Nombre de tienda con al menos una letra; nombre y apellido para personas.
- Número de cuenta de 6 a 17 números y que no sea un solo dígito repetido.
- Documento según el tipo (CC, NIT/RUT con dígito tras guion, pasaporte con letras).
- Dirección con el analizador del 1 a 1, que dice exactamente qué falta.
- Archivos revisados por su contenido real (PDF, JPG o PNG), máximo 5 MB.
- Pegar o seleccionar texto no cierra el modal.

---

## Endpoints

| Acción | Llamada |
| --- | --- |
| Leer | `GET metodoGenerico?coleccion=Tienda`, `Users`, `TiendaDocuments`, `CuentasBancarias`, `Ciudades` |
| Crear tienda | `insertarGenerico` en `Tienda` (forma completa, en orden) y `actualizarGenerico` en `Users` con `{ tiendas }` |
| Editar tienda | `actualizarGenerico` en `Tienda` con solo remitente, dirección y ciudad, en el orden de la tienda |
| Mis datos | `actualizarGenerico` en `Users` |
| Foto y banner | Sube a `archivos-muro-compartido` y `actualizarGenerico` (o `insertarGenerico`) en `PerfilesUsuario` |
| Volver a la predeterminada | `actualizarGenerico` en `PerfilesUsuario` con `avatarUrl: ''` y/o `bannerUrl: ''` |
| Documento | Sube a S3 y `insertarGenerico` en `TiendaDocuments` |
| Cuenta bancaria | `insertarGenerico` en `CuentasBancarias` (un registro con su `CuentasBancoTienda`) |

`GUARDADO_ACTIVO` en el repositorio apaga la escritura (lanza `GuardadoBloqueadoError`).

---

## Historial de cambios

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-11-18 | Yadilton Torres | Rediseño: tiendas con Id y `TipoTienda`, nombre único, detalle con autorizados y `.zip`, cuentas con el formato de siempre |
| 2026-11-19 | Yadilton Torres | Cuentas inactivas ocultas y bloqueo al registrarlas de nuevo |
| 2026-11-19 | Yadilton Torres | Modales con revisión en vivo, ✓ y resumen; listas siempre hacia abajo |
| 2026-11-19 | Yadilton Torres | Naranja solo en la rueda de avance; botones y pasos en azul, "en revisión" en ámbar |
| 2026-11-23 | Yadilton Torres | Crear tienda con la forma y el orden de Administración (`armarTiendaNueva`); ejemplo de teléfono «Ej: 300 123 4567» |
| 2026-11-24 | Yadilton Torres | Se trae de `main` el rediseño «La Red» y «Cambiar contraseña»; colores fijos pasan a `--lg-*` (modo noche) |
| 2026-11-24 | Yadilton Torres | «Usar la predeterminada» para foto y banner |
| 2026-11-24 | Yadilton Torres | Organización en `helpers/`, `models/` y `styles/`; pruebas de reglas, repositorio y detalle de tienda (61 en total) |

---

## Observaciones

- **Bucket de documentos:** van a `logigho-documentos-tienda-prod` **aunque se pruebe en PreProd**.
- Las tiendas de práctica que quedaron sin Id (versión anterior de Mi perfil) no afectan; Seguimiento les asigna uno al guardarlas.
- Opcionales sin hacer: largo de la cuenta por banco (Bancolombia 11) y dígito de verificación del NIT.
