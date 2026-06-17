---

## Autor: Adalberto González

Fecha creacion: 2026-05-07  
Estado: desarrollo  
Tipo: vista  

# Vista: WalletComponent

**Selector:** `app-wallet`  
**Ubicación:** `src/app/views/tienda/wallet`  
**Acceso:** Autenticado | Rol: `CEO`, `Administrador`, `Controller`, o tienda propia

---

## ¿Qué hace?

Panel central de gestión financiera de las tiendas. Muestra saldos, movimientos, liquidaciones, novedades, recargas, facturación y apalancamiento. Permite filtrar toda la información por tienda y abre modales de desembolso, novedades, facturación, tarjetas y apalancamiento.

---

## Ruta

| Ruta | Guard | Parámetros de URL |
| --- | --- | --- |
| `/app/tienda/wallet` | `AuthGuard` + validación por rol en `ngOnInit` | — |

---

## Propiedades clave

> Solo las no obvias por su nombre.

| Propiedad | Tipo | Descripción |
| --- | --- | --- |
| `selectedTienda` | `string` | Id de la tienda activa en el filtro; `'Todos'` por defecto |
| `decompressResume` | `any[]` | Cache del último `ResumenSaldoActualLogigho` descomprimido; compartido entre `fetchStoreResume` y `fetchTableDataApalancamiento` — si se llaman en paralelo pueden pisarse |
| `totalSaldoDisponible` | `number` | Bloquea el modal de desembolso si es negativo |
| `estadoWalletActivo` | `boolean` | Habilita/deshabilita el botón de desembolso según configuración de la tienda |
| `columnTitlesRetiros` | `Record<string, string>` | Mapea nombres de campo de BD a etiquetas legibles para la tabla de retiros |
| `isRoleAuthorized` | `boolean` | `true` si el rol es CEO, Administrador o Controller; controla visibilidad de secciones |

---

## Servicios y endpoints

| Servicio | Endpoint | Cuándo |
| --- | --- | --- |
| `ConsumoGenericoService` | `GET TransaccionesWalletLogigho` + filtro `Tienda` | `ngOnInit` y al cambiar tienda |
| `ConsumoGenericoService` | `GET ResumenSaldoActualLogigho` + filtro `Tienda` | `ngOnInit` y al cambiar tienda — saldo total |
| `ConsumoGenericoService` | `GET LiquidacionesLogigho` + filtro `NombreTienda` | `ngOnInit` y al cambiar tienda |
| `ConsumoGenericoService` | `GET NovedadesLogigho` + filtro `NombreTienda` | `ngOnInit` y al cerrar modal de novedades |
| `ConsumoGenericoService` | `GET Facturacion` + filtro `NombreTienda` | `ngOnInit` y al cerrar modal de facturación |
| `ConsumoGenericoService` | `GET Apalancamiento` + filtro `Tienda` | `ngOnInit` y al cambiar tienda |

---

## Flujo principal

```
ngOnInit()
  -> validarAccesoWallet()
  -> fetchTableDataLiquidacionesLogigho()
  -> fetchTableDataTransaccionesWalletLogigho()
  -> fetchTableDataNovedadesLogigho()
  -> fetchTableDataRecargas()
  -> fetchTableDataFacturacion()
  -> fetchStoreResume()
  -> fetchTableDataApalancamiento()

onTiendaSelect(value)
  -> applyFilters()          // filtra en memoria sin nueva petición
  -> recarga tablas con el filtro de tienda seleccionada

closeModal() / closeModalNovedades() / closeModalFacturacion()
  -> refresca solo los datos afectados + saldo
```

---

## Historial de cambios

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-05-07 | Adalberto González | Verificación del componente hijo Novedades. Sin cambios en Wallet. |
| 2026-06-16 | Adalberto González | Actualización de la tabla de retiros: eliminación de columnas obsoletas, incorporación de ComisionBanco, visualización de comprobantes y personalización de títulos de columnas. |

---

## Observaciones

- `applyFilters()` filtra en memoria desde `rowsMemory*` — no hace nueva petición. Solo se recarga desde la BD al cambiar tienda o cerrar un modal.
- `openModal()` valida que `totalSaldoDisponible >= 0` antes de abrir el desembolso — no modificar ese guard.
- `parseMonto()` en `fetchTableDataTransaccionesWalletLogigho` existe porque MongoDB devuelve los montos como string con formato `"$  16.455.000,00"`, no como número.
- El filtro `arrayTiendas.includes("Todas")` es el patrón correcto para usuarios con acceso global.
