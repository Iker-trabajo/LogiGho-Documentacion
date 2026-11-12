# Comercial: Panel comercial, Seguimiento y Accesos en LOGI — noviembre 2026

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
