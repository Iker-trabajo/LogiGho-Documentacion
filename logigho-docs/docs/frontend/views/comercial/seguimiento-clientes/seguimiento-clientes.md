---

## Autor: Yadilton Torres

Fecha creacion: 2026-11-11  
Estado: activo  
Tipo: vista

# Vista: Seguimiento de clientes

**Selector:** `app-seguimiento-tiendas`  
**Ubicación:** `src/app/views/comercial/seguimiento-clientes/`  
**Ruta:** `/app/comercial/clientes`  
**Acceso:** roles `CEO`, `COO`, `Administrador`, `Desarrollador` y `Comercial`

---

## ¿Qué hace?

Sigue a cada interesado desde el registro hasta su tienda activa, y administra tiendas, roles y servicios. Tiene dos vistas:

- **Personas:** cada persona de `RegisterContact` y cada cuenta de `Users`, con su proceso, sus datos, sus tiendas y sus roles.
- **Tiendas:** cada tienda de `Tienda`, con su avance, sus usuarios, sus documentos, sus cuentas bancarias, sus servicios y sus datos internos.

Para el proceso comercial completo (canal, estrategia, asesora, notas) está el [Panel comercial](../panel-comercial/panel-comercial.md).

---

## Archivos

```
comercial/seguimiento-clientes/
  seguimiento-tiendas.component.ts / .html / .spec.ts
  helpers/seguimiento-tiendas.rules.ts (+ .spec.ts)
  models/seguimiento-tiendas.models.ts
  repository/seguimiento-tiendas.repository.ts (+ .spec.ts)
  styles/seguimiento-tiendas.scss, seguimiento-tiendas.mobile.scss
```

| Archivo | Responsabilidad |
| --- | --- |
| `seguimiento-tiendas.component.ts` / `.html` | Pantalla: pestañas, filtros, tablas, paneles de persona y tienda, Excel |
| `helpers/seguimiento-tiendas.rules.ts` | Reglas puras: comparar tiendas (por Id o nombre normalizado), leer el remitente, remitente sugerido, sucursal y tipo «del Panel», fechas de creación, pasos, documentos requeridos y cuentas |
| `models/seguimiento-tiendas.models.ts` | `PersonaSeguimiento`, `TiendaSeguimiento`, `UsuarioSeguimiento`, `DatosInternosTienda`, `RemitenteTienda`, filtros |
| `repository/seguimiento-tiendas.repository.ts` | Lee las colecciones y guarda tiendas y usuarios |
| `styles/seguimiento-tiendas.scss` / `.mobile.scss` | Estilos |

### Pruebas (37)

| Archivo | Qué prueba |
| --- | --- |
| `seguimiento-tiendas.component.spec.ts` | Notas de origen, tipos de negocio, resumen con el Id, lo que se manda a guardar y validaciones del remitente |
| `repository/seguimiento-tiendas.repository.spec.ts` | Cruce con dueños y registros, datos «del Panel», fuente que falla, Id una sola vez, completar sin pisar y orden de los campos |
| `helpers/seguimiento-tiendas.rules.spec.ts` | Comparar tiendas, remitente, remitente sugerido, «del Panel», fechas, documentos, pasos y cuentas |

---

## Roles

| Constante | Roles | Qué permite |
| --- | --- | --- |
| `ROLES_PERMITIDOS` | CEO, COO, Administrador, Desarrollador, Comercial | Entrar a la pantalla |
| `ROLES_CONTROL_TOTAL` | CEO, COO, Administrador, Desarrollador | Ver y cambiar Wallet y Apalancamiento |

Los roles que Comercial puede poner o quitar a una cuenta son los de `ROLES_DE_COMERCIAL` (`views/comercial/roles-comercial.ts`): Backoffice, BPO, Carga Pedidos, Dropshipper, Externo, GLOBAL ACCOUNT, Logigho, Pedidos Masivo, Proveedor, Red social, Tienda y TiendaNW.

### Quién asigna qué roles

| Quién | Ve | Asigna | Tiendas |
| --- | --- | --- | --- |
| CEO, Administrador, Desarrollador | Todos los roles | **Todos** | Agrega y quita |
| COO (sin los de arriba) | Todos los roles | Solo `ROLES_DE_COMERCIAL`; los demás salen bloqueados (`asignaComoComercial`) | Agrega y quita |
| Comercial (sin los de arriba) | Solo `ROLES_DE_COMERCIAL` (`soloComercial`) | Solo `ROLES_DE_COMERCIAL` | No |

- A Comercial, quien tenga algún rol fuera de su lista le sale **solo para ver** (`esSoloLectura`).
- Al guardar, los roles que no se pueden tocar **se conservan**.
- En el [Panel comercial](../panel-comercial/panel-comercial.md) todos (incluso CEO) asignan solo `ROLES_DE_COMERCIAL`.
- El modal viejo "Asignar acceso" toma los roles del backend (`autenticacion/roles-asignables`).

---

## Datos que lee

`metodoGenerico?coleccion=` `Tienda`, `Users`, `TiendaDocuments`, `CuentasBancarias` y `RegisterContact`, todas las páginas.

---

## Pantalla

### Encabezado

- Pestañas **Personas** y **Tiendas**.
- Contador **"nuevos esta semana"** e interruptor **Ocultar pruebas** al lado.
- Botón verde **Exportar Excel** con lo que dejan los filtros (`Clientes_<día>.xlsx` o `Tiendas_<día>.xlsx`, día de Colombia). La columna **País** muestra "Colombia" u "Otro país".

### Filtros

Fila fija **"Mostrar solo"** (siempre visible):

| Vista | Botones |
| --- | --- |
| Personas | Sin teléfono |
| Tiendas | Con Wallet, Con apalancamiento, Por asignar, Sin remitente |

En las dos: [Filtro de fechas](../../../components/filtro-fechas.md) (**Registro** en Personas, **Contrato** en Tiendas) y **Limpiar filtros**. Las listas desplegables usan la [Lista de opciones](../../../components/lista-opciones.md).

### Tabla

Pie azul pegado a la tabla con el selector de registros por página (5, 10, 15 o 25; siempre blanco) y el paginador.

---

## Panel de persona

En este orden:

1. **Encabezado:** correo y teléfono para copiar y WhatsApp aparte. Si la cuenta entra con otro correo, sale "Entra a LOGI con …".
2. **Proceso comercial:** pasos (llenó el formulario, contactado por Comercial, creó su cuenta, tiene tienda). El estado de contacto se cambia con una lista; si lo puso el sistema (por ejemplo "Registrado en plataforma") se muestra como un dato más: *"Lo asigna la plataforma automáticamente"*.
3. **Datos de la persona.**
4. **Tiendas en LOGI.**
5. **Roles en LOGI.**
6. **Registro inicial** (plegado).

## Panel de tienda

En este orden: **Avance**, **Datos de la tienda** (ID LOGI para copiar), **Usuarios que tienen esta tienda**, **Activación y servicios**, **Datos internos**, **Remitente**, **Documentos requeridos** y **Cuentas bancarias**.

### Activación y servicios

| Interruptor | Comportamiento |
| --- | --- |
| Tienda activa | Se prende y se apaga |
| Wallet | Solo con control total. **Una vez activo en la tienda no se puede desactivar** (dañaría procesos): sale bloqueado con *"Activo, no se puede desactivar"* |
| Apalancamiento | Igual que Wallet |

Si Wallet o Apalancamiento estaban apagados, se pueden prender y, antes de guardar, volver a apagar. Después de guardar quedan bloqueados.

### Datos internos

Tipo de tienda, sucursal, integración, fulfillment, ecosistema y fecha de contrato.

### Lo que se llena solo

Si la tienda no tiene el dato, al abrirla se llena con lo que ya existe y lleva una nota junto al título; la nota se quita en cuanto alguien cambia el dato:

| Campo | De dónde | Nota |
| --- | --- | --- |
| Nombre del remitente | `RegisterContact.userName` del dueño (o del correo de la tienda); si no, el nombre de la cuenta. Nunca el `username` | «del registro» / «de la cuenta» |
| Teléfono del remitente | Del registro o de la cuenta del dueño | «del registro» / «de la cuenta» |
| Dirección del remitente | `direccionRecoleccion` del Panel comercial | «del Panel» |
| Sucursal | `sucursalInter` del Panel comercial | «del Panel» |
| Tipo de negocio | `businessType` del Panel, con los nombres de la tienda (`propio` → `Propia`, `Dropshipper` → `Dropshipping`; «Otro» guarda lo escrito) | «del Panel» |

El remitente que se llenó solo **no se manda** si nadie lo cambió. La sucursal y el tipo sí quedan escritos al guardar los datos internos (es el mismo dato real).

### Id y campos que le faltan a la tienda

- **Tienda sin Id** (creada antes del 19 de noviembre de 2026): al guardar se le pone el siguiente número, **una sola vez**. El resumen lo avisa primero: *«Id de la tienda: Sin Id → Se asigna el siguiente número»*.
- **Campos que le falten** a la tienda (por ejemplo `TiendaBase`): se agregan con su valor por defecto en el mismo guardado, **sin pisar** nada que ya tenga, aunque esté vacío.
- Todo lo que se envía va en el orden de la [Forma de la tienda](../../../core/forma-tienda.md). Lo que no existía queda al final (así funciona MongoDB).

### Guardar

- Al cerrar, al pasar con ↑ o ↓ a otra fila, o con "Ver persona" desde una tienda, si hay cambios aparece **"¿Salir sin guardar?"** con la lista de cambios.
- Al guardar también se muestra lo que cambió.

| Qué | Llamada |
| --- | --- |
| Tienda (estado, wallet, apalancamiento, datos internos, remitente) | `actualizarGenerico` en `metodoGenerico?coleccion=Tienda` |
| Usuario (tiendas y roles) | `actualizarGenerico` en `metodoGenerico?coleccion=Users` |

El remitente se guarda en un solo texto: `Nombre: … | Direccion: … | Telefono: …`. `GUARDADO_ACTIVO` en el repositorio apaga la escritura.

---

## Historial de cambios

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-11-11 | Yadilton Torres | Paneles de persona y tienda reorganizados, copiar en tablas y paneles, "¿Salir sin guardar?" |
| 2026-11-11 | Yadilton Torres | Filtros en fila fija "Mostrar solo"; filtro de fechas con atajos; se quita el filtro de roles |
| 2026-11-11 | Yadilton Torres | Wallet y apalancamiento no se desactivan una vez activos; se quita Transportadora de datos internos |
| 2026-11-11 | Yadilton Torres | "País" en lugar de "Es de Colombia"; Exportar Excel; pie azul |
| 2026-11-19 | Yadilton Torres | Se documenta quién asigna qué roles (control total, COO y Comercial) |
| 2026-11-20 | Yadilton Torres | Remitente vacío con el nombre del registro y el teléfono del registro o la cuenta, con su nota |
| 2026-11-23 | Yadilton Torres | Dirección, sucursal y tipo vacíos «del Panel»; notas en la línea del título; completa campos faltantes (`TiendaBase`) sin pisar y en orden |
| 2026-11-24 | Yadilton Torres | Colores del tema «La Red»; reglas a `helpers/`; pruebas de pantalla, repositorio y reglas (37) |

---

## Observaciones

- **Asignar acceso** (PR 640) vive en la pantalla vieja de Clientes (`clientes.component.*`); falta pasarlo a esta vista.
- **Aviso «No se pudo cargar: Roles»:** el servicio `/roles` del backend bloquea por CORS las peticiones desde `localhost`. Es del backend; lo demás carga igual.
- El resumen de Clientes muestra la fecha de contrato como `2025-06-15`; el Panel la muestra como `15/06/2025`.
- Los documentos de la tienda se suben desde Mi perfil (`views/tienda/perfil`): el archivo va al bucket S3 `logigho-documentos-tienda-prod` con el nombre `{IdTienda}_{Detalle}_{TipoDocumento}` y el registro a `TiendaDocuments` (`Validado: false`). Aquí solo se ven y se cuentan ("0 de 4 validados"). El registro no guarda la ruta del archivo (`Documento` va vacío) y dos documentos con el mismo detalle y tipo se reemplazan en S3.
