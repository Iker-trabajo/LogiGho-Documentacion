# Comercial: Panel comercial, Seguimiento y Accesos en LOGI — noviembre 2026

## [2026-11-24] — Tema «La Red» y módulos organizados con pruebas

### Cambios
- Se trae de `main` el rediseño «La Red». Los colores fijos y las variables `--logigho-*` que quedaban en el Panel y en Clientes (botones de tipo de negocio, cuenta inactiva, notas «del Panel») pasan a `--lg-*`.
- **Panel comercial:** reglas puras a `helpers/panel-comercial.rules.ts`; `panel-comercial.cuentas.spec.ts` se separa en `panel-comercial.component.spec.ts`, `repository/panel-comercial.repository.spec.ts` y la base falsa `helpers/panel-comercial.repository.fake.ts`.
- **Clientes:** reglas puras del repositorio a `helpers/seguimiento-tiendas.rules.ts`; `seguimiento-remitente.spec.ts` pasa a `repository/seguimiento-tiendas.repository.spec.ts` (ampliado) y hay un spec nuevo de la pantalla.
- La pantalla y lo que se guarda no cambian.

### Pruebas
- Panel 46 (antes 28), con reglas y guard. Clientes 37 (antes 5).

---

## [2026-11-23] — «Su negocio» por tienda, fecha de contrato y listas que no tapan

### Nuevas funcionalidades
- **Fecha de contrato** (`Tienda.FechaContrato`) en «Su negocio» del Panel, de la tienda elegida: en el detalle y en Editar. Se guarda como en Clientes (`AAAA-MM-DD`) y se ve `dd/mm/aaaa`.
- **«Su negocio» por tienda:** tipo, fulfillment, dirección y fecha son de cada tienda. Elegir tienda no es un cambio, se vuelve a consultar la tienda y cada una tiene su borrador por `_id`; al guardar escribe solo lo suyo en su `_id`. «Despacha desde» y «Paquetes» son de la persona.
- **Clientes → Tienda:** dirección del remitente, sucursal y tipo vacíos se llenan con lo del Panel (nota «del Panel»); se completan los campos que le falten a la tienda (por ejemplo `TiendaBase`) sin pisar nada y en orden. Ver [Forma de la tienda](../frontend/core/forma-tienda.md).

### Cambios
- El selector de tienda sale siempre que haya tiendas en LOGI, aunque sea una.
- Tarjeta «Más de 150 paquetes/mes» (antes «100+»).
- Las notas de origen («del registro», «del Panel») van en la línea del título y ya no descuadran los campos.

### Correcciones
- La lista abierta quedaba tapada por la sección de abajo: la lista marca su etiqueta (`lista-opciones--abierta`) y la sección sube.
- Al abrir una lista la pantalla saltaba: ya no se mueve el scroll.
- La lista de tienda no se cerraba al elegir: estaba dentro de un `<label>`.
- Los botones de tipo de negocio heredaban la tarjeta «¿En qué etapa van?» (misma clase `pc-tipos`): ahora tienen su clase.

### Pruebas
- Probado en PreProd con paco (pandal5 y paracaida): fecha de contrato Panel ↔ Clientes, cada tienda con su fecha, Estrategia y Paquetes con tabla y filtro; todo restaurado al final. pandal5 recibió su **Id 157148** al guardarla en Clientes (único).

---

## [2026-11-20] — Datos que ya existen, listas nuevas y varias tiendas

### Nuevas funcionalidades
- **Panel → Editar:** teléfono y cédula vacíos con los de la cuenta; tienda y dirección con los de la tienda; con nota y sin contar como cambio.
- **Sucursal:** la tarjeta de una tienda sin sucursal muestra la del proceso («del proceso»); al guardarla se escribe también en `Tienda.Sucursal` de las tiendas que no tienen.
- **Canal de contacto «Otro (escribir cuál)»**, obligatorio escribirlo.
- **Estrategia LOGI:** Sin definir, A (0 a 10), B (10 a 50), C (50 a 150), D (más de 150), Estrategia comercial y Especial; las categorías viejas se leen como el nivel nuevo.
- **Paquetes al mes:** Menos de 50, De 50 a 150 y Más de 150 (también en Accesos en LOGI).
- **Varios tipos de negocio** en `businessType`, separados por coma (`shared/utils/tipos-negocio.ts`); Accesos en LOGI pasa todos a `TipoTienda`.
- **Clientes → Tienda:** remitente vacío con el nombre del registro del dueño y el teléfono del registro o la cuenta.

### Correcciones
- El constructor de dirección no dejaba hacer scroll cuando se abría sobre un modal.

---

## [2026-11-11] — Panel comercial, Seguimiento de clientes y formulario Accesos en LOGI

### Nuevas funcionalidades
- **Panel comercial** (`/app/comercial/panel-comercial`, COO, CEO y Desarrollador): une `RegisterContact`, `Users` y `Tienda` por correo; detalle, edición del proceso comercial, roles de Comercial, Excel y filtros. Reemplaza la hoja "Registro de Tiendas".
- Personas con cuenta pero sin registro salen como "No llenó el formulario" y al editarlas se les crea su `RegisterContact`.
- **Formulario Accesos en LOGI**: 4 pasos, todos los campos obligatorios, ciudad con buscador desplegable. Sale obligatorio a quien tiene solo el rol Red social hasta que lo complete (`fechaAccesos`), antes de la guía. También desde el menú de la cuenta.
- **Notificaciones** `NUEVO_REGISTRO_ACCESOS` (a Comercial y COO) y `REGISTRO_RESPONDIDO` (a la persona).
- Componentes compartidos **Lista de opciones** y **Filtro de fechas** (atajos y rango).

### Cambios
- Seguimiento de clientes: paneles reorganizados, filtros en fila fija "Mostrar solo", filtro de fechas, "País", Exportar Excel, pie azul; se quitan el filtro de roles y el campo Transportadora.
- Wallet y apalancamiento no se pueden desactivar una vez activos.
- Fechas en UTC en la base y en hora de Colombia en pantalla, filtros y Excel.
- PreProd: las llamadas se reparten entre la API HTTP (`g3iz6qk3f0`) y la REST (`8u5isw5u30`) con `RUTAS_REST`; auditoría apagada en PreProd.

### Correcciones
- Las listas desplegables se cierran al presionar fuera también dentro de los paneles.
- El selector de registros por página ya no desaparece sobre el pie azul.
- La carga del usuario tiene respaldo por `metodoGenerico` cuando `/users` no lo encuentra.

### Documentación
- [Panel comercial](../frontend/views/comercial/panel-comercial/panel-comercial.md), [Campos de RegisterContact](../frontend/views/comercial/panel-comercial/registercontact-campos.md), [Seguimiento de clientes](../frontend/views/comercial/seguimiento-clientes/seguimiento-clientes.md).
- [Accesos en LOGI](../frontend/components/accesos-logi.md), [Lista de opciones](../frontend/components/lista-opciones.md), [Filtro de fechas](../frontend/components/filtro-fechas.md).
- [PreProd: API HTTP y API REST](../guias/apis-preprod-http-rest.md).
- Actualizados: [Servicio de Notificaciones](../frontend/core/notificaciones-service.md), [Default Layout](../frontend/layout/default-layout.md), [Default Header](../frontend/layout/default-header.md).

### Pendiente
- Backend: aceptar en `RegisterContact` los 6 campos del formulario y los 9 del panel; crear en la REST las rutas que faltan; revisar `/users` de PreProd.
- Registrar el Panel comercial en la colección `module` y quitar la línea TEMPORAL del layout.
- Limpiar en la base los correos repetidos y con mayúsculas de `RegisterContact`.
- Historial de notas (`historialComercial`).
- Pasar "Asignar acceso" (PR 640) a Seguimiento.
