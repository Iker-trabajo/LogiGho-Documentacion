---

## Autor: Adalberto González

Fecha creacion: 2026-05-07  
Estado: desarrollo  
Tipo: componente  

# Componente: CreacionCuentaBancariaComponent

**Selector:** `app-creacioncuentabancaria`  
**Ubicación:** `src/app/components/creacioncuentabancaria`  
**Acceso:** Autenticado | usado desde `PerfilComponent`

---

## ¿Qué hace?

Modal para registrar una cuenta bancaria asociada a una tienda. Carga las tiendas asignadas al usuario desde `sessionStorage`, permite seleccionar tienda, banco, tipo de documento y tipo de cuenta mediante dropdowns buscables, valida todos los campos y guarda el registro en la colección `CuentasBancarias`.

---

## Decoradores

| Decorador | Nombre | Tipo | Descripción |
| --- | --- | --- | --- |
| `@Input` | `isOpen` | `boolean` | Controla si el modal está visible |
| `@Output` | `closeEvent` | `EventEmitter<void>` | Notifica al padre que el modal se cerró |

---

## Propiedades clave

| Propiedad | Tipo | Descripción |
| --- | --- | --- |
| `isLoading` | `boolean` | Bloquea el botón para evitar envíos duplicados |
| `tiendas` | `any[]` | Cargado desde la API según `tiendas_asignadas` del sessionStorage |
| `bancos` | `Banco[]` | Lista estática de bancos colombianos |
| `tiposDocumento` | `TipoDocumento[]` | Lista estática (CC, TI, CE, NIT, RUT, PA) |
| `tiposCuenta` | `TipoCuenta[]` | Lista estática (corriente, ahorros, billetera virtual) |

---

## Servicios y endpoints

| Servicio | Método | Endpoint | Cuándo |
| --- | --- | --- | --- |
| `ConsumoGenericoService` | `consultarGenerico()` | `GET Tienda` + filtro `NombreTienda` | `ngOnInit` — carga el dropdown de tiendas |
| `ConsumoGenericoService` | `insertarGenerico()` | `POST CuentasBancarias` | Al confirmar el formulario |
| `DecompressionService` | `decompressZstd()` | — | Descomprime respuesta con `mcomp: '2'` |

---

## Flujo principal

```
ngOnInit()
  -> fetchTableDataTienda()   // carga tiendas filtradas por tiendas_asignadas

createAccount()
  -> guard isLoading          // evita doble clic
  -> form.valid               // valida los 7 campos requeridos
  -> construye CuentaBancaria + CuentaBancoTienda + ApiRequest
  -> insertarGenerico()       // POST a CuentasBancarias
      -> close()
```

---

## Historial de cambios

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-05-07 | Adalberto González | Refactor completo: fix filtro "Todas", `firstValueFrom` + `mcomp:'2'` + `decompressZstd`, `Validators.required` en los 7 campos, bloqueo de doble clic con `isLoading`, 4 dropdowns buscables (Tienda, Banco, Tipo Documento, Tipo Cuenta), SCSS con paleta `var(--logigho-*)` |

---

## Observaciones

- El filtro de tiendas usa `arrayTiendas.includes("Todas")` — mismo patrón que `NovedadesComponent` y `WalletComponent`. Sin este check el dropdown queda vacío para usuarios con acceso global.
- `bancos`, `tiposDocumento` y `tiposCuenta` son listas estáticas en el TS — no vienen de la BD. Si se necesita hacerlas dinámicas, requeriría un endpoint nuevo.
- La estructura que se guarda en MongoDB anida `CuentaBancaria` dentro de `CuentaBancoTienda` dentro de `ApiRequest` — respetar ese modelo al leer o modificar registros.
