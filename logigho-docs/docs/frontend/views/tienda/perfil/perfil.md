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

| Archivo | Responsabilidad |
| --- | --- |
| `perfil.component.ts` / `.html` | Pantalla: avance, tiendas, documentos, cuentas y los modales para crear o editar |
| `perfil.model.ts` | `TiendaPerfil`, `CuentaPerfil`, `DocumentoPerfil`, `PropietarioPerfil`, `PasoPerfil`, `ModalPerfil` |
| `repository/perfil.repository.ts` | Escrituras: crear y editar tienda, mis datos, fotos, documentos (S3 + `TiendaDocuments`) y cuentas |
| `components/detalle-tienda/` | "Detalle y documentos" de una tienda: pestañas Datos, Documentos tienda, una por autorizado y "+ Autorizado" |
| `perfil.component.scss` / `.mobile.scss` | Estilos de escritorio y celular (≤768 px) |
| `perfil.validaciones.spec.ts` | Pruebas de los modales y de las cuentas inactivas |

Componentes compartidos que usa: [Lista de opciones](../../../components/lista-opciones.md), `app-confirmar-cambios`, `app-paginador`, `app-direccion-estandarizada` (Armar dirección del 1 a 1) y la directiva `appModalTeclado`.

---

## Pantalla

### Avance

- **Rueda de porcentaje:** cuenta los 5 pasos (mis datos, crear tienda, foto y banner, documentos y cuenta bancaria). Va en naranja mientras falta algo y pasa a verde al 100 %. El mismo avance se publica en el encabezado (`AvancePerfilService`).
- **Siguiente paso:** tarjeta azul con el paso que falta y su botón.
- **Pasos:** el actual en azul con pulso, los completos en verde y los pendientes en gris.

!!! note "Colores"
    El naranja de la marca se usa **solo en la rueda de avance**. Botones y pasos van en azul (`--logigho-azul`) y lo que está "en revisión" o "sin ID" en ámbar suave (`#fef3c7` / `#92400e`), igual que en el resto de la app.

### Tiendas

- **Crear tienda:** nombre, correo (fijo, el de la cuenta), tipo de negocio (varios y "Otro: cuál"), teléfono, ciudad y dirección. Ciudad y dirección son obligatorias.
- La tienda **nace inactiva** y asignada al usuario. Comercial la activa.
- **Id:** lo calcula el front como el más alto más 1, igual que Administración → Tiendas. Si una tienda vieja no tiene Id, Seguimiento se lo pone al guardarla.
- **Nombre único** en toda la plataforma, sin importar mayúsculas, tildes ni espacios.
- **`TipoTienda`:** se guarda como en producción (`Propia`, `Dropshipping`, `Proveedor` u otro).
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

Todas las listas de los modales usan `[siempreAbajo]="true"`: abren hacia abajo y, si no hay espacio, el modal baja lo necesario. La de ciudades (más de mil) se pinta por tandas. En celular salen como hoja inferior.

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
| Crear tienda | `insertarGenerico` en `Tienda` y `actualizarGenerico` en `Users` con `{ tiendas }` |
| Editar tienda | `actualizarGenerico` en `Tienda` con solo los cambios |
| Mis datos | `actualizarGenerico` en `Users` |
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

---

## Observaciones

- **Bucket de documentos:** van a `logigho-documentos-tienda-prod` **aunque se pruebe en PreProd**.
- Las tiendas de práctica que quedaron sin Id (versión anterior de Mi perfil) no afectan; Seguimiento les asigna uno al guardarlas.
- Opcionales sin hacer: largo de la cuenta por banco (Bancolombia 11) y dígito de verificación del NIT.
