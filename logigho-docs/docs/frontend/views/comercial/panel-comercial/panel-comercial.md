---

## Autor: Yadilton Torres

Fecha creacion: 2026-11-11  
Estado: activo  
Tipo: vista

# Vista: Panel comercial

**Selector:** `app-panel-comercial`  
**Ubicación:** `src/app/views/comercial/panel-comercial/`  
**Ruta:** `/app/comercial/panel-comercial`  
**Acceso:** roles `COO`, `CEO` y `Desarrollador` (`panelComercialGuard`)

---

## ¿Qué hace?

Es la pantalla del jefe de comercio para llevar el proceso de cada persona que se registra en LOGI: desde que llena el registro hasta que tiene su tienda activa. Reemplaza la hoja de Google "LOGI · Registro de Tiendas (Formulario)", donde antes se llevaban el canal, la estrategia, la asesora, la sucursal de Interrapidísimo y las notas.

Junta en una sola fila por persona lo que está en tres colecciones:

| Colección | Qué aporta |
| --- | --- |
| `RegisterContact` | Registro inicial, respuestas del formulario [Accesos en LOGI](../../../components/accesos-logi.md) y el proceso comercial |
| `Users` | Cuenta en LOGI: usuario, roles, tiendas, nombre, teléfono, cédula y autorización de datos |
| `Tienda` | Tiendas de la cuenta: ID LOGI, sucursal, estado, wallet y apalancamiento |

También muestra a quienes **tienen cuenta pero nunca llenaron el registro**: salen como **"No llenó el formulario"** y se pueden editar igual (ver [Crear el registro](#crear-el-registro-de-quien-no-lleno-el-formulario)).

---

## Archivos

| Archivo | Responsabilidad |
| --- | --- |
| `panel-comercial.component.ts` / `.html` | Pantalla: tabla, filtros, detalle, edición, Excel |
| `panel-comercial.guard.ts` | Deja entrar solo a `ROLES_PANEL_COMERCIAL = ['COO', 'CEO', 'Desarrollador']`; si no, lleva al Muro compartido |
| `models/panel-comercial.models.ts` | `PersonaPanel`, `TiendaPanel`, `ProcesoComercial`, `EdicionPersona` y listas fijas (`CANALES`, `ESTRATEGIAS`, `ESTADOS_PROCESO`) |
| `repository/panel-comercial.repository.ts` | Único punto que llama al backend: lee las tres colecciones, las une, crea y guarda |
| `styles/panel-comercial.scss` | Estilos de la pantalla |
| `../roles-comercial.ts` | `ROLES_DE_COMERCIAL`: los roles que Comercial puede poner o quitar (compartido con Seguimiento) |

Componentes compartidos que usa: [Lista de opciones](../../../components/lista-opciones.md), [Filtro de fechas](../../../components/filtro-fechas.md), `app-paginador`, `app-confirmar-cambios` y la directiva `appPanelRedimensionable` (panel lateral que se agranda arrastrando el borde).

---

## Cómo se unen las colecciones

La conexión es **por correo**. El `_id` no sirve porque cada colección tiene el suyo, y casi ningún `RegisterContact` tiene `userId`.

```
RegisterContact.userId ─► Users.cognitoId        (si existe)
RegisterContact.email  ─► Users.email            (minúsculas, sin espacios)
Users.tiendas[]        ─► Tienda.NombreTienda    (nombre normalizado, sin tildes)
   si por nombre no encuentra ninguna:
Users.email            ─► Tienda.Correo          (respaldo)
```

- Cuentas de `Users` cuyo correo no está en `RegisterContact` se agregan con `sinRegistro = true` y estado **"No llenó el formulario"**.
- `Users.nombre` casi nunca está lleno: si falta, el nombre se toma de `username`.
- Orden: primero quien llenó el formulario más reciente (`fechaAccesos`), luego por `fechaRegistro`.
- Se marcan como prueba (`esPrueba`) los correos con `+test` o `prueba`. El interruptor **"Ocultar pruebas"** los quita.
- Las tres colecciones se leen completas, página por página (`NumeroPaginas`). Si `Users` o `Tienda` fallan, se muestra lo demás y un aviso con lo que no cargó.

---

## Pantalla

### Encabezado y pestañas

- Pestañas **Todos**, **Con formulario** y **Sin formulario**, con su cantidad.
- **Ocultar pruebas** al lado de las pestañas.
- Buscador: nombre, correo, cédula, teléfono, tienda, ID LOGI, sucursal, dirección, asesora y nota (sin tildes ni mayúsculas).
- Botón verde **Exportar Excel**: descarga lo que dejan los filtros. El archivo lleva el día de Colombia en el nombre. Incluye la columna **País**.

### Filtros

| Filtro | Valores |
| --- | --- |
| Tipo de negocio | Tienda propia, Dropshipper, Proveedor, Otro y **Sin dato** (valor interno `__sin_tipo__`) |
| Despacha desde | Ciudades que existen en los registros |
| Cómo se enteró | Fuente del registro |
| Estado del proceso | Pendiente, Seguimiento, Activada, Inactiva y "Sin estado" |
| Estrategia | `ESTRATEGIAS` |
| Fecha de registro | [Filtro de fechas](../../../components/filtro-fechas.md): Hoy, Ayer, Últimos 7 días, Últimos 30 días, Este mes, Mes pasado o rango |

Cada opción muestra cuántas personas tiene. **Limpiar** borra todos los filtros.

### Tabla

- Correo, teléfono, cédula, ID LOGI y sucursal tienen botón para **copiar**; al copiar sale un ✓.
- Pie azul pegado a la tabla con el selector de registros por página (siempre en blanco, también después de elegir 10 o 25) y el paginador.

### Detalle de una persona

Panel lateral, en este orden:

1. **Encabezado:** nombre, correo y teléfono para copiar, y botón de WhatsApp aparte.
2. **Proceso comercial:** estado de proceso, estado de contacto, canal, estrategia, asesora, sucursal Inter y fecha de accesos. La **nota** va en su propio recuadro naranja.
3. **Su negocio:** tienda, tipo, despacha desde, fulfillment, paquetes al mes y dirección de recolección. El título dice la fecha del formulario o "No llenó el formulario".
4. **Datos de la persona:** nombre, cédula, teléfono y habeas data. Lo que viene de `Users` lleva la etiqueta "de la cuenta".
5. **Cuenta y tiendas en LOGI:** usuario con el que entra, roles y una tarjeta por tienda (ID LOGI y sucursal para copiar, activa, wallet, apalancamiento).
6. **Registro inicial** (plegado): fecha, cómo se enteró, cuándo comenzaría, ventas y país.

---

## Edición

Mismo orden que el detalle, en bloques: **Proceso**, **Su negocio**, **Datos de la persona** y **Roles**.

### Validaciones

Solo se revisa lo que cambió. El campo con error se pone en rojo y no deja guardar.

| Campo | Regla |
| --- | --- |
| Nombre, ciudad, asesora | Solo letras |
| Cédula | De 5 a 12 números |
| Teléfono | 10 números |
| Sucursal Inter | Solo números |
| Dirección | Debe tener calle y número |
| Fecha de accesos | No puede ser futura |

### Reglas al guardar

- **Solo se envían los campos que cambiaron** (`cambios(antes, despues)`), más `fechaActualizacionComercial` y `actualizadoPor` (correo de quien guardó).
- **Tienda registrada bloqueada:** si la persona ya tiene tiendas en LOGI, el nombre de la tienda sale con candado y no se envía, porque está conectado con muchas otras cosas.
- **Estado de contacto:** desde aquí solo puede quedar en `No Contactado` o `Contactado`. Los estados que pone el sistema (por ejemplo **"Registrado en plataforma"**) se muestran solo para leer.
- **Sucursal Inter:** si está vacía, se propone la sucursal de su tienda; solo se guarda si la cambian.
- **Teléfono** se guarda como número.
- **Fecha de accesos** se guarda como la medianoche de Colombia en UTC.
- **Roles:** solo se ponen o quitan los de `ROLES_DE_COMERCIAL`. Los demás roles que tenga la cuenta se ven bloqueados y se conservan tal cual (`guardarRoles`).

### Salir sin guardar

Con Cancelar, la X, Esc o clic fuera aparece **"¿Salir sin guardar?"** con la lista de cambios. Al cerrar o recargar la pestaña, el navegador también pregunta.

### Crear el registro de quien no llenó el formulario

Si la persona tiene `sinRegistro = true`, al guardar se le **crea su `RegisterContact`** (`crear()`, ruta `registerContact`) con todo lo que hay en pantalla, incluido lo que vino de su cuenta, estado `No Contactado`, `fechaRegistro`, `fechaActualizacionComercial` y `actualizadoPor`. Luego se recarga la lista. El `username` no se guarda como nombre.

---

## Endpoints

| Acción | Llamada |
| --- | --- |
| Leer | `GET metodoGenerico?coleccion=RegisterContact`, `...Users`, `...Tienda` (todas las páginas) |
| Guardar persona | `actualizarGenerico` en `metodoGenerico?coleccion=RegisterContact` con solo los cambios |
| Crear registro | `insertarGenericoST` en `registerContact` |
| Guardar roles | `actualizarGenerico` en `metodoGenerico?coleccion=Users` con `{ roles }` |

`GUARDADO_ACTIVO` en el repositorio apaga la escritura sin tocar la pantalla (lanza `GuardadoBloqueadoError`).

---

## Campos de `RegisterContact`

Ver [Campos de RegisterContact](registercontact-campos.md).

---

## Fechas

En la base se guardan en UTC. En pantalla, en los filtros y en el Excel se muestran en hora de Colombia (`America/Bogota`). Una cuenta con `createdAt` guardado como texto (`"12/3/2026"`) se lee como día/mes/año.

---

## Historial de cambios

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-11-11 | Yadilton Torres | Creación del Panel comercial: une RegisterContact, Users y Tienda; detalle, edición, roles, Excel y filtros |
| 2026-11-11 | Yadilton Torres | Personas sin registro ("No llenó el formulario") editables: se les crea su RegisterContact |
| 2026-11-11 | Yadilton Torres | Filtro de fecha de registro; se quitan los filtros de asesora y paquetes al mes; se quita Puntuación |

---

## Observaciones

- **El menú todavía no viene de la colección `Modulo`:** la ruta está agregada a mano en `default-layout.component.ts` (línea marcada `TEMPORAL`). Cuando se registre el módulo, se quita esa línea.
- **Correos repetidos y con mayúsculas:** hay personas con más de un `RegisterContact` y correos con mayúsculas o espacios. El panel los muestra todos; conviene limpiarlos en la base.
- **`Tienda.Correo` se repite mucho** (un mismo correo puede estar en decenas de tiendas). El respaldo por correo solo se usa si por nombre no se encontró ninguna tienda.
- **La nota reemplaza a la anterior:** no hay historial de notas todavía.
- **País:** hoy solo se sabe si es de Colombia (`isColombian`); se muestra "Colombia" u "Otro país".
- El guard solo controla la página: los datos los debe proteger el backend.
