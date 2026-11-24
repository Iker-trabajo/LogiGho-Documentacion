---

## Autor: Yadilton Torres

Fecha creacion: 2026-11-23  
Estado: activo  
Tipo: utilidad compartida

# Utilidad: Forma de la tienda (campos, orden e Id)

**Ubicación:** `src/app/shared/utils/forma-tienda.ts` y `src/app/shared/utils/siguiente-id-tienda.ts`  
**Pruebas:** `forma-tienda.spec.ts`  
**Dónde se usa:** [Mi perfil](../views/tienda/perfil/perfil.md) (crear tienda), [Accesos en LOGI](../components/accesos-logi.md) (primera tienda), [Seguimiento de clientes](../views/comercial/seguimiento-clientes/seguimiento-clientes.md) y [Panel comercial](../views/comercial/panel-comercial/panel-comercial.md) (al actualizar)

---

## ¿Qué hace?

Hace que **toda tienda nueva** quede con los mismos campos y en el mismo orden con que nace en Administración → Tiendas, y que al **actualizar** una tienda los campos vayan en ese orden y no falte ninguno.

Antes, Mi perfil y Accesos en LOGI creaban la tienda con otro orden, sin `Fulfillment`, `FechaContrato`, `Ecosistema` y, las más viejas, sin `Id` ni `TiendaBase`.

---

## Forma de referencia

La de una tienda creada en Administración → Tiendas (por ejemplo GAME CENTER WORLD, Id 157174):

| # | Campo | Por defecto al crear |
| --- | --- | --- |
| 1 | `Id` | El siguiente número (ver abajo) |
| 2 | `Sucursal` | `''` |
| 3 | `Transportadora` | `''` |
| 4 | `Remitente` | `Nombre: … \| Direccion: … \| Telefono: …` |
| 5 | `Integracion` | `''` |
| 6 | `NombreTienda` | — (obligatorio) |
| 7 | `Fulfillment` | `''` |
| 8 | `TipoTienda` | `[]` |
| 9 | `Correo` | — (obligatorio) |
| 10 | `FechaContrato` | `''` |
| 11 | `Estado` | `INACTIVO` |
| 12 | `Ecosistema` | `''` |
| 13 | `TiendaBase` | `{ Id: null, NombreTienda: 'Sin Tienda Base' }` |
| 14 | `EstadoWallet` | `false` |
| 15 | `EstadoApalancamiento` | `false` |
| 16 | `Direccion` | `null` |
| 17 | `CodigoDane` | `null` |

`TiendaBase` conecta una tienda con otra "madre" (en Administración se elige en «Seleccionar Tienda Base»). Si no depende de ninguna, se guarda «Sin Tienda Base».

---

## Funciones

| Función | Para qué |
| --- | --- |
| `armarTiendaNueva(datos)` | Tienda nueva con los 17 campos en orden; lo que no venga toma su valor por defecto |
| `ordenarComoTienda(datos)` | Pone las llaves de lo que se envía en el orden de la forma (lo desconocido va al final) |
| `camposQueFaltan(tienda, cambios)` | Campos que la tienda **no tiene** (la llave no existe), con su valor por defecto. No incluye el `Id` ni lo que ya viene en `cambios`, y **nunca pisa** un campo que exista, aunque esté vacío |
| `revisarNuevaTienda(…)` | Antes de crear: revisa que el nombre no exista (sin mayúsculas, tildes ni espacios) y calcula el Id |
| `siguienteIdTienda(…)` | El `Id` numérico más alto de todas las tiendas, más 1 (la misma regla de Administración) |

---

## El orden al actualizar

MongoDB deja cada campo que **ya existe** en su lugar y agrega al **final** los que no existían. Por eso:

- Una tienda **nueva** nace completa y en orden: al editarla nada se mueve.
- A una tienda **vieja** a la que le faltan campos (por ejemplo `Id` o `TiendaBase`), esos campos se agregan al final, pero en el orden correcto entre ellos.
- Reordenar una tienda vieja exigiría reescribirla completa, lo que es riesgoso con datos reales: no se hace.

El orden no cambia cómo funciona nada: Pedidos, Documentos Tiendas y las búsquedas usan el nombre del campo.

---

## El Id de la tienda

Pedidos y Documentos Tiendas encuentran la tienda **por su `Id`**. Una tienda sin `Id` existe, pero esas pantallas no la unen con sus pedidos ni documentos.

| Dónde | ¿Pone el Id? |
| --- | --- |
| Crear en Mi perfil, Accesos en LOGI o Administración | Sí, al crearla |
| Guardar en Clientes → Tienda una tienda sin Id | Sí, una sola vez; el resumen lo avisa: *«Id de la tienda: Sin Id → Se asigna el siguiente número»* |
| Editar en el Panel comercial o en Mi perfil | No |

Las tiendas creadas desde Mi perfil o el formulario **antes del 19 de noviembre de 2026** nacieron sin `Id`; se completan guardándolas en Clientes.

---

## Historial de cambios

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-11-19 | Yadilton Torres | `revisarNuevaTienda` y `siguienteIdTienda`: nombre único e Id al crear desde Mi perfil y Accesos en LOGI |
| 2026-11-23 | Yadilton Torres | `forma-tienda.ts`: forma y orden de Administración al crear; Clientes completa lo que falte sin pisar nada |

---

## Observaciones

- **Tiendas con `"Id": "N/A"`** (por ejemplo YADI1 en producción): ese texto no lo escribe este código; viene de Administración → Tiendas, que muestra «N/A» en los campos vacíos. Esas tiendas no tienen un Id real.
