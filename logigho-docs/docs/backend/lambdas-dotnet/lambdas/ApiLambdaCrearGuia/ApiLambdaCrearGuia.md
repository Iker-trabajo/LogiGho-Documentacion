

## Autor: Iker Acevedo
Fecha creacion: 2026-09-14

Estado: preprod

## Lambda: ApiLambdaCrearGuia


**Accionador:** API Gateway

**AOT:** No

---

## ¿Qué hace?

Crea guías de forma individual (no por lote) para las transportadoras integradas (Interrapidísimo, Envia, XCargo, Servientrega, D2E, TCC). Recibe un arreglo de pedidos, inserta el/los registro(s) en `CargaPedido` (o `CargaPedidoCrm` si `Origen == "CRM"`) y por cada uno acciona el caso de uso de la transportadora correspondiente: crea el preenvío en la API de la transportadora, dispara la generación de etiqueta (llamada a `/creacionEtiqueta`, no descarga el PDF acá — a diferencia de `ApiLambdaCargarGuias`) y guarda el pedido en `PedidosInter`. A diferencia de `ApiLambdaCargarGuias`, procesa los pedidos de forma síncrona uno a uno (no hay lotes en paralelo) y no maneja buckets S3 directamente.

---

## Accionador

| Método | Ruta | Autenticacion |
| ------ | ---- | ---- |
| `POST` | API Gateway — `ApiLambdaCrearGuia` | Bearer token (Cognito) |

---

## Request

```json
[
  {
    "TRANSPORTADORA": "TCC",
    "NOMBRE TIENDA": "BTK MAN",
    "DIRECCION ORIGEN": "CALLE 43 B # 82 -62 BARRIO LA AMERICA",
    "ASESOR": "Alejandra Osorio",
    "TELEFONO REMITENTE": "3112222222",
    "CIUDAD ORIGEN": "05001000",
    "IDENTIFICACION (OPCIONAL)": "3205591719",
    "NOMBRE COMPLETO": "Luz dary guerra arroyo",
    "DIRECCION": "CALLE 78B # 84 - 22 BARRIO ROBLEDO EL DIAMANTE",
    "TELEFONO": "3205591719",
    "CIUDAD": "05001000",
    "PESO": "1",
    "LARGO": "10",
    "ALTO": "10",
    "ANCHO": "10",
    "VALOR DECLARADO": "320200",
    "APLICA CONTRA PAGO": "SI",
    "OBSERVACIONES": "Camisetas Adultos PNEW - REF: TALLA XL:12 | TALLA L:4 | TALLA M:1",
    "DICE CONTENER": "Camisetas Adultos PNEW",
    "IdTienda": "156783",
    "TIPO ETIQUETA": "Sticker"
  }
]
```

El body es un arreglo JSON, aunque se envíe un solo pedido. El token de autenticación va en el header:
```
Token: eyJraWQi...
```

---

## Response

### Exitoso

```json
{
  "Resultado": ["603974894"],
  "Error": false,
  "Mensaje": null
}
```

`Resultado` es un arreglo con el número de guía (`NumeroPreenvio`) por cada pedido procesado, en el mismo orden del request. Si un pedido individual falla, su posición en el arreglo trae `"Error {mensaje}"` en vez del número de guía — el lote no se detiene por un pedido fallido.

### Errores

| Código | Cuándo |
| ------ | ------ |
| `500` | Excepción no controlada en el handler principal (ej. JSON del request mal formado) |

---

## Flujo interno

```
FunctionHandler (Function.cs)
  -> Lee TRANSPORTADORA del primer item del request para elegir la colección de ciudades
     (CiudadesInter / CiudadesXCargo / CiudadesServientrega / CiudadesTCC)
  -> En paralelo: ProcesarColeccionesRepositoryAsync (TipoEntregaInter, ciudades, Tienda, Productos)
                  + InsertarJArrayColeccionAsync(coleccionCarga, jsonObjectRequest)
  -> Por cada pedido, en orden (síncrono, sin paralelismo), según TRANSPORTADORA:

     [INTERRAPIDISIMO] CargaInterUseCase.procesarRegistroCarga
     [ENVIA]           CargaEnviaUseCase.procesarRegistroCarga
     [D2 / XCargo]     CargaXCargoUseCase.procesarRegistroCarga
     [SERVIENTREGA]    CargaServientregaUseCase.procesarRegistroCarga
     [D2E]             CargaD2EUseCase.procesarRegistroCarga

     [TCC] CargaTccUseCase.procesarRegistroCarga
       -> Valida cobertura: CIUDAD ORIGEN y CIUDAD (DANE8) en CiudadesTCC con Estado=ACTIVA
       -> Define paquetería/mensajería según PESO (>5kg = paquetería)
       -> API TCC: tarifas/v6/consultarliquidacion (OBLIGATORIA, antes del despacho, fail-fast)
       -> API TCC: remesas/grabardespacho7 -> obtiene numeroremesa
       -> item["NumeroPreenvio"] = remesa (parseado a long)
       -> Dispara /creacionEtiqueta (opción 9, no espera el PDF acá)
       -> MongoDB: actualiza CargaPedido (Estado=Cargado)
       -> MongoDB: inserta en PedidosInter (Trayecto = tipoenvio de la liquidación)

  -> Responde con el arreglo de resultados (guía o error) por pedido
```

---

## Transportadora TCC (CargaTccUseCase)

Integración con TCC (`ApiLambdaCrearGuia.Aplicacion.CasosUso.Carga.CargaTccUseCase`). Endpoint base: `URL_SERVICIO_TCC`. `OpcionConsumoTcc = 10` en el switch de `Generico.cs` de esta lambda (distinto del `9` en `ApiLambdaCargarGuias`/`ApiLambdaCrearEtiquetaManual` — el número de opción depende de cuántos servicios tiene mapeados cada `Generico.cs`, no es un valor fijo entre lambdas).

La lógica de negocio (paquetería/mensajería, cobertura por ciudad, liquidación obligatoria con fail-fast, bypass de sandbox, suma de sobreflete agrupada, `Trayecto`) es la **misma** que en `ApiLambdaCargarGuias` — ver el detalle completo en [ApiLambdaCargarGuias → Transportadora TCC](../ApiLambdaCargarGuias/ApiLambdaCargarGuias.md#transportadora-tcc-cargatccusecase), para no duplicar la explicación.

Diferencias puntuales de esta lambda frente a `ApiLambdaCargarGuias`:

- **No descarga ni re-aloja el PDF del rótulo.** Después del despacho exitoso, arma un `EtiquetaRequest` y llama a `/creacionEtiqueta` (opción de consumo `9`, la lambda de creación de etiqueta interna) — el PDF se resuelve en otro punto del flujo, no acá.
- **No usa `procesarAlarmas`** (validación de ventas duplicadas / % devoluciones) — esa validación es exclusiva de `ApiLambdaCargarGuias`, que procesa cargas masivas donde el riesgo de duplicados es mayor.
- El límite del loop `CANTIDAD STOCK {i}` es `i <= 12` (en `ApiLambdaCargarGuias` es `i <= 18`) — sigue la misma convención que ya tenían Inter/Servientrega en cada lambda.
- No tiene el ajuste de `PrimerNombreRemitente`/`RazonSocialRemitente` con fallback a `item["Tienda"]` como sí lo tiene esta lambda puntualmente (`item["NOMBRE TIENDA"] ?? item["Tienda"]`) — cubre el caso en que el pedido individual no trae `NOMBRE TIENDA`.

---

## Variables de entorno

| Variable | Descripción | Valores |
| -------- | ----------- | ------- |
| `URL_SERVICIO_INTER` | URL base de la API de Interrapidísimo | URL |
| `URL_SERVICIO_AWS` | URL base de servicios internos AWS | URL |
| `CADENA_CONEXION` | Cadena de conexión MongoDB (encriptada AES) | String encriptado |
| `DATABASE_NAME` | Nombre de la base de datos MongoDB | `"LogighoDB"` |
| `URL_SERVICIO_TCC` | URL base de la API de TCC | `https://testsomos.tcc.com.co/api/clientes/` (sandbox) |
| `TCC_ACCESS_TOKEN` | AccessToken de la API TCC (encriptado AES) | String encriptado |
| `TCC_NIT` | NIT/clave del cliente TCC (encriptado AES) | String encriptado |
| `TCC_CUENTA_PAQUETERIA` | Cuenta TCC para envíos > 5kg | `"1485100"` |
| `TCC_CUENTA_MENSAJERIA` | Cuenta TCC para envíos <= 5kg | `"5625200"` |
| `TCC_PERMITIR_LIQUIDACION_FALLIDA` | Bypass del fallo de recaudo en sandbox. **Solo en preprod, nunca en producción** | `"true"` / ausente (= `false`) |

---

## Dependencias externas

| Servicio | Uso |
| -------- | --- |
| `API Interrapidísimo` | Crear preenvío |
| `API Envia` | Crear guías Envia |
| `API XCargo / D2` | Crear guías XCargo y D2 |
| `API Servientrega` | Crear guías Servientrega |
| `API D2E` | Crear guías D2E |
| `API TCC` | Consultar liquidación (`tarifas/v6/consultarliquidacion`) y crear despacho (`remesas/grabardespacho7`) |
| `Lambda ApiLambdaCrearEtiquetaManual` (`/creacionEtiqueta`) | Generación del PDF de la guía, disparada tras el despacho exitoso de cada transportadora |

---

## Historial de cambios

| Fecha | Autor | Cambio |
| ----- | ----- | ------ |
| 2026-09-01 | Iker Acevedo | Integración TCC (HU 3.1 — creación individual de guía): `CargaTccUseCase`, entidades en `Dominio/Entidades/TCC`, liquidación previa obligatoria con fail-fast, mapeo paquetería/mensajería por peso, validación de cobertura por ciudad. |
| 2026-09-03 | Iker Acevedo | Fix `NumeroPreenvio` de TCC: se guarda como `long`, no `string`, consistente con el resto de transportadoras. |
| 2026-09-03 | Iker Acevedo | Fix suma de sobreflete TCC: se agrupa por `idconcepto` y se toma el valor máximo de cada grupo (TCC duplica conceptos con el mismo id). |
| 2026-09-03 | Iker Acevedo | `TCC_PERMITIR_LIQUIDACION_FALLIDA`: bypass controlado por variable de entorno para probar en preprod pedidos con recaudo (sandbox de TCC no soporta recaudo). Ausente/`false` por defecto. |
| 2026-09-14 | Iker Acevedo | `Trayecto` en `PedidosInter` para TCC se llena con `total.tipoenvio` de la respuesta de liquidación, reutilizando la clasificación que TCC ya calcula. |

---

## Observaciones

- Esta lambda procesa los pedidos **síncronamente uno a uno**, a diferencia de `ApiLambdaCargarGuias` que usa lotes de 3 en paralelo — apropiada para creación individual, no para cargas masivas.
- El PDF de la guía no se resuelve acá para ninguna transportadora: se delega a `/creacionEtiqueta`, que internamente invoca los mismos casos de uso `GenerarEtiqueta` que expone `ApiLambdaCrearEtiquetaManual`.
- `OpcionConsumoTcc` (el número de "opción de consumo" que arma la URL en `Generico.cs`) **no es el mismo valor en las tres lambdas TCC** — cada `Generico.cs` tiene su propio switch con sus propios servicios ya mapeados antes de agregar TCC. Al modificar el switch de `Generico.cs` de cualquiera de las tres lambdas, verificar el número real de la opción TCC en ese archivo puntual antes de copiar el valor de otra lambda.
