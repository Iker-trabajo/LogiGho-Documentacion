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

Es un Panel central de gestión financiera de las tiendas. Muestra saldos, movimientos, liquidaciones, novedades, recargas, facturación y apalancamiento. Permite filtrar toda la información por tienda mediante un selector, y desde aquí se abren los modales de desembolso, novedades, facturación, tarjetas y apalancamiento.

---

## Ruta

| Ruta | Guard | Parámetros de URL |
| --- | --- | --- |
| `/app/tienda/wallet` | `AuthGuard` + validación por rol en `ngOnInit` | — |

---

## Propiedades clave

| Propiedad | Tipo | Descripción |
| --- | --- | --- |
| `selectedTienda` | `string` | Id de la tienda seleccionada en el filtro; `'Todos'` por defecto |
| `tiendaOptions` | `any[]` | Lista de tiendas cargadas desde la BD, incluyendo `EstadoWallet` y `EstadoApalancamiento` |
| `decompressResume` | `any[]` | Cache del último `ResumenSaldoActualLogigho` descomprimido; lo reusan `actualizarWalletPorTienda()` y filtros |
| `totalSaldoDisponible` | `number` | Suma de saldos de las tiendas visibles; bloquea el modal de desembolso si es negativo |
| `totalApalancamientoDisponible` | `number` | Cupo de apalancamiento disponible para la tienda seleccionada |
| `isRoleAuthorized` | `boolean` | `true` si el usuario tiene rol CEO, Administrador o Controller; controla visibilidad de secciones |
| `estadoWalletActivo` | `boolean` | Habilita/deshabilita botón de desembolso según configuración de la tienda |
| `walletResumen` | `any[]` | Lista de tiendas con su saldo individual, usada para el widget de resumen visual |

---

## Servicios y endpoints

| Servicio | Método | Endpoint | Cuándo |
| --- | --- | --- | --- |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET Tienda` + filtro `NombreTienda` | `ngOnInit` — carga el selector de tiendas |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET ResumenSaldoActualLogigho` + filtro `Tienda` | `ngOnInit` y al cambiar tienda — saldo total |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET TransaccionesWalletLogigho` + filtro `Tienda` | `ngOnInit` y al cambiar tienda |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET Apalancamiento` + filtro `Tienda` | `ngOnInit` y al cambiar tienda |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET LiquidacionesLogigho` + filtro `NombreTienda` | `ngOnInit` y al cambiar tienda |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET NovedadesLogigho` + filtro `NombreTienda` | `ngOnInit` y al cerrar modal de novedades |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET Facturacion` + filtro `NombreTienda` | `ngOnInit` y al cerrar modal de facturación |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET TarjetaTienda` + `IdTienda` | Al abrir modal de tarjeta |

---

## Flujo principal

```
ngOnInit()
  -> validarAccesoWallet()       // redirige si no tiene rol
  -> cargarTasaCambio()
  -> fetchTableDataLiquidacionesLogigho()
  -> fetchTableDataTransaccionesWalletLogigho()
  -> fetchTableDataNovedadesLogigho()
  -> fetchTableDataRecargas()
  -> fetchTableDataFacturacion()
  -> fetchStoreResume()          // saldo total
  -> fetchTableDataApalancamiento()
  -> onTiendaSelect('Todos')     // inicializa filtros y wallet resumen

onTiendaSelect(value)
  -> actualizarWalletPorTienda() // recalcula widget de saldos
  -> recarga todas las tablas con el filtro de tienda seleccionada
  -> applyFilters()              // filtra filas en memoria sin nueva petición

closeModal() / closeModalNovedades() / closeModalFacturacion()
  -> refresca solo los datos afectados + saldo
```

---

## Historial de cambios

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-05-07 | Adalberto González | Verificaión de el componete hijo [Novedades](/frontend/components/novedades-component/) `No hubo cambios en el componente Wallet`. |

---

## Observaciones

- El filtro `arrayTiendas.includes("Todas")` en los métodos de fetch es el patrón correcto para usuarios con acceso global.
- `applyFilters()` filtra en memoria desde `rowsMemory*`; no hace nueva petición. Solo se recarga desde la BD al cambiar tienda o cerrar un modal.
- `openModal()` (desembolso) valida que `totalSaldoDisponible >= 0` antes de abrir — no modificar ese guard.
- `decompressResume` es compartido entre `fetchStoreResume` y `fetchTableDataApalancamiento`; si ambos se llaman en paralelo pueden pisarse.
