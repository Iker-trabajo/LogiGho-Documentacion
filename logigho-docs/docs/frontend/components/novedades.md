---

## Autor: Adalberto González

Fecha creacion: 2026-05-07  
Estado: desarrollo  
Tipo: componente

# Componente: NovedadesComponent

**Selector:** `app-novedades`  
**Ubicación:** `src/app/components/novedades`  
**Acceso:** Autenticado | Rol: administrador wallet

---

## ¿Qué hace?

Modal para registrar novedades de crédito/débito en el wallet de una tienda. Carga las tiendas asignadas al usuario desde `sessionStorage`, permite seleccionarla con un dropdown buscable, valida todos los campos antes de guardar y actualiza automáticamente el resumen de saldo de la tienda afectada.

---

## Decoradores

| Decorador | Nombre | Tipo | Descripción |
| --- | --- | --- | --- |
| `@Input` | `isOpen` | `boolean` | Controla si el modal está visible |
| `@Input` | `montoDisponible` | `string` | Saldo actual de la tienda seleccionada |
| `@Output` | `closeEvent` | `EventEmitter<void>` | Notifica al padre que el modal se cerró |
| `@Output` | `refreshDataEvent` | `EventEmitter<void>` | Notifica al padre que debe refrescar datos |

---

## Propiedades clave

| Propiedad | Tipo | Descripción |
| --- | --- | --- |
| `isLoading` | `boolean` | Bloquea el botón para evitar envíos duplicados |
| `tiendasFiltradas` | `any[]` | Lista del dropdown filtrada en tiempo real |
| `tiendaSeleccionadaNombre` | `string` | Nombre visible de la tienda elegida |
| `idTransaccionResumen` | `string` | `_id` del documento en `ResumenSaldoActualLogigho` que se actualiza al guardar |

---

## Servicios y endpoints

| Servicio | Método | Endpoint | Cuándo |
| --- | --- | --- | --- |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET Tienda` + filtro `NombreTienda` | `ngOnInit` — carga el dropdown |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET ResumenSaldoActualLogigho` + filtro `Tienda` | Antes de guardar — obtiene saldo e `_id` |
| `ConsumoGenericoService` | `insertarGenerico()` | `POST NovedadesLogigho` | Al confirmar el formulario |
| `ConsumoGenericoService` | `actualizarGenerico()` | `PUT ResumenSaldoActualLogigho` | Tras insertar — actualiza el saldo |
| `DecompressionService` | `decompressZstd()` | — | Descomprime respuestas con `mcomp: '2'` |

---

## Flujo principal

```
ngOnInit()
  -> fetchTableDataTienda()   // carga dropdown de tiendas (filtra por tiendas_asignadas)
  -> fetchTableData()         // carga cuentas bancarias (no visible en UI actual)

createNovedad()
  -> guard isLoading (evita doble clic)
  -> fetchStoreResume()       // obtiene saldo actual e idTransaccionResumen
  -> insertarGenerico()       // guarda en NovedadesLogigho
      -> actualizarGenerico() // actualiza TotalSaldo en ResumenSaldoActualLogigho
      -> close()
```

---

## Historial de cambios

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-05-07 | Adalberto González | Refactor completo: dropdown buscable para Tienda, layout 2 columnas, validación required en todos los campos, bloqueo de doble clic con `isLoading`, fix filtro "Todas" para usuarios con "Todas" asignada como tienda en el sessionStorage |

---

## Observaciones

- El campo `Monto` se guarda como string formateado en pesos colombianos (`"$  31.787.000,00"`), no como número. Es el formato histórico de la colección — no cambiar.
- El filtro de tiendas usa `arrayTiendas.includes("Todas")` para usuarios con acceso global; sin este check el dropdown queda vacío para esos usuarios.
- `fetchTableData()` (cuentas bancarias) se llama en `ngOnInit` pero no se usa en el template actual — posible deuda técnica.
