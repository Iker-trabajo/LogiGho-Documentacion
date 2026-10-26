---
autor: Iker Acevedo
fecha_creacion: 2026-10-25
ultima_actualizacion: 2026-10-25
estado: vigente
---

# Probar la API de guías en PreProd

Pasos para dejar la API funcionando en PreProd y probarla de punta a punta con Interrapidísimo. **Los ejecuta Iker**: la IA no escribe en AWS ni en la base (regla del proyecto). Los scripts viven en `Lambdas_NET_LogiGho` → `pipelines/servicio-publico/scripts/`.

> **Aviso.** Crear una guía llama a la Interrapidísimo **real** con la URL que tiene `ApiLambdaCargarGuias` en PreProd. La guía de prueba es una guía de verdad en Inter (con datos ficticios): hay que anularla o avisar después. Decidido por Iker (2026-10-24).

---

## 0. Qué cambió desde la última vez que se aplicó Terraform

| Qué | Dónde |
| --- | --- |
| Quinta función `ApiLambdaGuiasCalculadoraPreProd` (cálculo del cupo), con permiso de invocación para la API, el worker y el reconciliador | Terraform |
| Alarmas `documento-ausente` y `fallida-por-reintentos`; la calculadora entra en las alarmas de errores y throttles | Terraform |
| Variables del worker: `OCULTAR_REMITENTE_ETIQUETA`, `TEXTOS_OCULTAR_ETIQUETA`, `FUNCION_CALCULADORA_CUPO` | Terraform solo al **crear** una función; las ya existentes las pone el script (Terraform ignora su entorno) |
| `URL_SERVICIO_AWS` pasa a `URL_AVISO_INVENTARIO`, **vacía** en PreProd (el paso de inventario se omite con registro y métrica) | script |
| WAF: PreProd sigue **sin** WAF | — |

---

## 1. Terraform (Iker)

```powershell
cd "C:\Users\Iker\Desktop\Logigho-repos\Terraform_Infraestructura_Logigho-guias\environments\preprod\guias"
terraform init
terraform plan -out guias-preprod5.tfplan
```

El plan debe mostrar **solo altas**: la función de cálculo, su rol y su grupo de registros, las políticas de invocación de los otros tres roles, las dos alarmas nuevas y la alarma de errores/throttles de la calculadora. **Nada que destruya** una función existente. Si aparece algo más, parar y revisar. Después:

```powershell
terraform apply guias-preprod5.tfplan
```

---

## 2. Código y variables (Iker)

Los cuatro zips salen de `verificar.sh` (carpeta `zips`) o de `dotnet lambda package` en cada proyecto: `ApiLambdaGuias`, `ApiLambdaGuiasWorker`, `ApiLambdaGuiasMcp`, `ApiLambdaGuiasCalculadora`.

```powershell
cd "C:\Users\Iker\Desktop\Logigho-repos\Lambdas_NET_LogiGho-guias\pipelines\servicio-publico\scripts"
.\desplegar-preprod.ps1 -Zips "<carpeta con los zips>"
```

Publica el código de cuatro funciones (el MCP queda pausado; `-ConMcp` lo incluye), copia las variables cifradas de `ApiLambdaCargarGuias` y pone `ENTORNO`, `FUNCION_CALCULADORA_CUPO`, `ESPERA_RESPUESTA_SEGUNDOS`, `URL_AVISO_INVENTARIO` (vacía) y las de la etiqueta. No imprime valores secretos.

---

## 3. Datos de prueba en Atlas PreProd (Iker)

Hoy `ApiCupoTiendas` está **vacía**, el cliente `integrador-prueba` no tiene todavía la tienda de prueba y el único producto de la tienda de prueba está **Inactivo**. Con `mongosh` sobre la base `LogighoDB` de Atlas PreProd:

**3.1 El cliente de prueba con la tienda `156783`** (el cliente ya existe; se reemplaza su documento):

```javascript
db.ApiClientes.replaceOne(
  { ClienteId: "54s6t64cb84b8np5p5cc63sfjj" },
  {
    ClienteId: "54s6t64cb84b8np5p5cc63sfjj",
    Nombre: "Integrador de prueba (PreProd)",
    Nit: "",
    Estado: "ACTIVO",
    Tiendas: [
      { IdTienda: 156783, Nombre: "Tienda de prueba 156783", Transportadoras: ["INTERRAPIDISIMO"], Productos: [] }
    ],
    IpsPermitidas: [],
    TopeLote: 10
  },
  { upsert: true }
)
```

`Productos: []` hereda lo que el catálogo de la tienda permite (RN-094). `TopeLote` es el máximo de guías por lote de ese cliente (50 si no se pone).

**3.2 El cupo de la tienda** (prepago, en `OBSERVAR`: calcula y registra pero no rechaza; sin límite diario de guías porque no se indica `GuiasPorDiaTienda`):

```javascript
db.ApiCupoTiendas.insertOne({
  _id: 156783,
  Tienda: "Tienda de prueba 156783",
  Modo: "PREPAGO",
  LimiteCredito: 0,
  Aplicacion: "OBSERVAR",
  Suspension: { Activa: false },
  Parametros: { MargenEstimacionPct: 25, TopeLote: 50 },
  Saldo: { Confiable: 1000000, CalculadoEn: new Date() },
  ExposicionExterna: 0,
  Reservado: 0,
  Base: 1000000,
  Disponible: 1000000,
  Version: 1
})
```

**3.3 El producto** `1371` (el único de la tienda) está `Inactivo` y una guía de Dropshipping necesita un producto activo del proveedor:

```javascript
db.Productos.updateOne({ idproducto: 1371 }, { $set: { estado: "Activo" } })
```

(Alternativa: usar otro producto activo de esa tienda y pasarlo con `-IdProducto` al script de pruebas.)

Los índices de las colecciones `Api*` los crea el propio código la primera vez que se usan.

**Lo que ya está verificado en PreProd (solo lectura):** la tienda `156783` existe, está `ACTIVO` y es `Dropshipping`; el proveedor del producto tiene DANE y dirección (origen de la guía); Medellín (`05001000`), Bogotá (`11001000`) y Cali (`76001000`) tienen cobertura de Inter; `CostosTransporteTienda` trae las filas de entrega y devolución de la tienda; `TarifasInter`, `AdministracionCostos` y `Marketing` están vacías (la calculadora lo tolera: sin trayecto conocido reserva con la columna de flete más cara).

---

## 4. Prueba de humo (Iker)

```powershell
cd "C:\Users\Iker\Desktop\Logigho-repos\Lambdas_NET_LogiGho-guias\pipelines\servicio-publico\scripts"
.\probar-preprod.ps1                      # solo token, salud y tiendas: no crea nada
.\probar-preprod.ps1 -CrearGuiasReales    # además: una guía, el mismo reintento y un lote de dos guías
```

Lee los secretos de `%USERPROFILE%\integrador-prueba-preprod-secreto.txt` y `mcp-api-key-preprod.txt` (no los imprime). Qué debe pasar:

| Paso | Resultado esperado |
| --- | --- |
| Salud y tiendas | `200`; la tienda `156783` aparece en `/tiendas` |
| Crear guía | `201` con `numeroGuia` si el worker termina antes de 20 s; si no, `202` y el script consulta `GET /solicitudes/{id}` hasta `CREADA` |
| Mismo reintento (misma clave y body) | `200` con la misma solicitud; **no** se crea otra guía en Inter |
| Lote con una guía mala (`peso` 0) | `202`: una `ACEPTADA` y una `RECHAZADA` con `rechazoId`, código y campo; `GET /lotes/{id}` termina con la buena en `CREADA` |

---

## 5. Qué mirar en Atlas después

| Colección | Qué debe verse |
| --- | --- |
| `ApiSolicitudesGuia` | La solicitud en `Creada`, con `NumeroGuia`, `PasosCompletados` y `Reserva.Origen = "Real"` (se ajustó con los valores de Inter). La del lote lleva `LoteId` |
| `ApiMovimientosCupo` | Una línea `Reserva` (estimada) y, si el valor real difiere, una `AjusteReserva` |
| `ApiCupoTiendas` | `Reservado` igual a la reserva viva de las guías sin liquidar |
| `CargaPedido` | Fila `Cargado`, `IdCarga` `"-1"`, `Origen` `"API"`, `ASESOR` `"Logigho"`; las del lote además `LoteApi` |
| `PedidosInter` | El pedido con `UrlGuia`; las del lote con `LoteApi` |
| `ApiLotesGuia` | El lote con un ítem aceptado y otro rechazado (con `RechazoId`) |

La etiqueta (`UrlGuia`) debe abrir **sin** «LOGIGHO S.A.S» en el remitente (si el sticker no se pudo transformar queda la original y la métrica `EtiquetaSinTransformar` lo marca).

---

## 6. Si algo no sale

| Síntoma | Causa probable |
| --- | --- |
| `503 CUPO_NO_DISPONIBLE` | La función de cálculo no responde o no tiene `CADENA_CONEXION`; revisar su registro y que el script la haya configurado |
| `422 CUPO_TARIFA_NO_RESUELTA` | La tienda no tiene flete configurado para Inter en `CostosTransporteTienda` |
| `403 TIENDA_NO_PERMITIDA` o `PRODUCTO_NO_PERMITIDO` | Falta el paso 3.1, o el producto sigue inactivo |
| `422 ORIGEN_NO_CONFIGURADO` | El proveedor del producto no tiene DANE o dirección |
| `403` del authorizer | Token vencido o scope insuficiente |
| `429`/`403` del gateway sin cuerpo de la API | Falta la API key (`x-api-key`) del plan de uso |
| La guía se queda en `RECIBIDA` | El worker no recibe mensajes o el cliente está bloqueado (la guía espera; métrica `GuiaEnEsperaClienteBloqueado`) |
| Alarma `documento-ausente` | Seguir [runbook-documento-ausente.md](runbook-documento-ausente.md) |

Desactivar la API de prueba es dejar `Estado: "BLOQUEADO"` en el documento del cliente: las guías aceptadas esperan y no se crean más.
