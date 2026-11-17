---

## Autor: Yadilton Torres

Fecha creacion: 2026-11-11  
Estado: activo  
Tipo: modelo de datos

# Colección `RegisterContact`: campos

`RegisterContact` guarda a cada persona que se registra en LOGI. Hoy tiene tres grupos de campos: los del registro inicial, los del formulario [Accesos en LOGI](../../../components/accesos-logi.md) y los del [Panel comercial](panel-comercial.md).

> La conexión con `Users` y `Tienda` es **por correo** (`email` en minúsculas). Casi ningún registro tiene `userId`.

---

## Registro inicial

| Campo | Tipo | Descripción |
| --- | --- | --- |
| `email` | `string` | Correo de la persona (llave para unir con `Users`) |
| `userId` | `string` | `cognitoId` de la cuenta, si existe |
| `userName` | `string` | Nombre completo |
| `phoneNumber` | `number` | Teléfono / WhatsApp (puede llegar como `{ $numberLong }`) |
| `city` | `string` | Ciudad desde donde despacha |
| `NombreComercio` | `string` | Nombre de la tienda |
| `businessType` | `string` | `propio`, `Dropshipper`, `Proveedor` u `Otro` |
| `source` | `string` | Cómo se enteró de LOGI |
| `comienzo` | `string` | Cuándo comenzaría |
| `ventasPorMes` | `string` | Ventas al mes |
| `isColombian` | `boolean` | Es de Colombia |
| `estado` | `string` | Estado de contacto: `No Contactado`, `Contactado` o uno que pone el sistema (por ejemplo `Registrado en plataforma`) |
| `fechaRegistro` | `string` (ISO, UTC) | Cuándo se registró |

## Formulario "Accesos en LOGI"

| Campo | Tipo | Descripción |
| --- | --- | --- |
| `documento` | `string` | Cédula o NIT del titular (5 a 12 números) |
| `direccionRecoleccion` | `string` | Dirección donde la transportadora recoge |
| `requiereFulfillment` | `boolean` | Quiere que LOGI guarde y despache su inventario |
| `paquetesMes` | `string` | `Menos de 50`, `50 a 100` o `Más de 100` |
| `aceptaHabeasData` | `boolean` | Autorizó el tratamiento de datos (Ley 1581 de 2012) |
| `fechaAccesos` | `string` (ISO, UTC) | Cuándo completó el formulario. **Si existe, el formulario ya no vuelve a salir** |

## Panel comercial

| Campo | Tipo | Descripción |
| --- | --- | --- |
| `canalContacto` | `string` | `CANALES`: Pauta, Comercial, Especial, Ecosistema B&C, Ecosistema Invicto, Ecosistema Moisés, Referido 13Store |
| `estrategia` | `string` | `ESTRATEGIAS`: Categoría B (10 a 50), C (50 a 150), D (0 a 10), CT-B (3 gratis), CT-C (3 gratis), Especial |
| `sucursalInter` | `string` | Sucursal de Interrapidísimo asignada |
| `asesora` | `string` | Asesora a cargo |
| `estadoProceso` | `string` | `Pendiente`, `Seguimiento`, `Activada` o `Inactiva` |
| `fechaEntregaAccesos` | `string` (ISO, UTC) | Día en que se entregaron los accesos (medianoche de Colombia) |
| `nota` | `string` | Nota libre (hasta 500 caracteres). Reemplaza la anterior |
| `actualizadoPor` | `string` | Correo de quien guardó el último cambio comercial |
| `fechaActualizacionComercial` | `string` (ISO, UTC) | Cuándo se guardó el último cambio comercial |

---

## Para backend

El backend debe aceptar los 6 campos del formulario y los 9 del panel en `RegisterContact` (insertar y actualizar por `metodoGenerico`).

## Historial de cambios

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-11-11 | Yadilton Torres | Campos del formulario Accesos en LOGI y del Panel comercial |
