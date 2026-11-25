---

## Autor: Yadilton Torres

Fecha creacion: 2026-11-19  
Estado: activo  
Tipo: modelo de datos

# Estado de las cuentas bancarias (activa / inactiva)

**Colección:** `CuentasBancarias`  
**Quién lo cambia:** [Panel comercial](../../comercial/panel-comercial/panel-comercial.md#cuentas-bancarias)  
**Quién lo respeta:** [Mi perfil](perfil.md#cuentas-bancarias)

---

## ¿Qué hace?

Permite que Comercial **inactive una cuenta bancaria** para que la persona ya no la vea en Mi perfil, sin borrarla. Así los desembolsos y novedades viejos que la usaron siguen mostrando a qué cuenta se pagó.

Para cambiar de cuenta: Comercial inactiva la vieja y la persona agrega la nueva. Si la persona necesita volver a usar la misma, Comercial la **reactiva** (no se crea un registro repetido).

---

## Campos

Van en **cada cuenta** de la lista `CuentasBancoTienda`, al lado de `IdTienda`:

```json
{
  "IdTienda": "10",
  "Tienda": "akazuki",
  "CuentaBancaria": { "Id": "…", "Banco": { "Nombre": "Bancolombia" }, "...": "..." },
  "Estado": "Inactiva",
  "FechaEstado": "2026-11-19T20:15:00.000Z",
  "UsuarioEstado": "comercial@logigho.com"
}
```

| Campo | Tipo | Descripción |
| --- | --- | --- |
| `Estado` | `"Activa"` \| `"Inactiva"` | **Opcional.** Sin el campo, la cuenta cuenta como **activa** (así no hubo que migrar las cuentas viejas) |
| `FechaEstado` | ISO en UTC | Cuándo se cambió por última vez |
| `UsuarioEstado` | correo en minúsculas | Quién la cambió (`sessionStorage.emailaddress`) |

Se compara sin mayúsculas: `Estado.trim().toLowerCase() === 'inactiva'`.

---

## Cómo se guarda

`PanelComercialRepository.cambiarEstadoCuenta(cuenta, activa)`:

1. Vuelve a leer `CuentasBancarias` justo antes de escribir, para no pisar una cuenta que se haya agregado mientras el panel estaba abierto.
2. Busca la cuenta por su posición; si se movió, la busca por número, banco y tienda. Si no la encuentra, **no escribe nada** y avisa *"No se encontró la cuenta: recarga el panel e inténtalo de nuevo."*
3. Cambia solo `Estado`, `FechaEstado` y `UsuarioEstado` de esa cuenta y conserva las demás.
4. `actualizarGenerico({ CuentasBancoTienda }, _id, 'metodoGenerico?coleccion=CuentasBancarias')`.

---

## Dónde se respeta

| Pantalla | Comportamiento |
| --- | --- |
| Mi perfil | No muestra las inactivas. No deja registrar de nuevo una cuenta inactiva (mismo número, banco y tienda): pide hablar con un asesor |
| Panel comercial | Muestra todas, activas primero, con etiqueta y botón Activar / Inactivar |
| Desembolso, Desembolso apalancamiento, Desembolso inversionista | **Todavía no filtra** |
| Novedades | **Todavía no filtra** |

---

## Historial de cambios

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-11-19 | Yadilton Torres | Creación del campo `Estado` y su uso en Mi perfil y Panel comercial. Probado en PreProd (inactivar y reactivar) |

---

## Observaciones

- **Pendiente:** que los 3 desembolsos y Novedades no ofrezcan cuentas inactivas, para no pagarle a una cuenta dada de baja.
- **Backend:** confirmar que `actualizarGenerico` actualiza solo `CuentasBancoTienda` y no reemplaza el registro completo.
- Una cuenta que se reactiva conserva los campos con `Estado: "Activa"`.
