---
autor: Iker Acevedo
fecha_creacion: 2026-10-22
ultima_actualizacion: 2026-10-22
estado: referencia
---

# Hallazgos en las lambdas actuales de guías

Revisión de solo lectura del 2026-10-22 sobre `ApiLambdaCrearGuia` (`CR`), `ApiLambdaCargarGuias` (`CG`), `ApiLambdaCrearEtiquetaManual`, `ServicioPublicoLogigho` y AWS prod. La API nueva los evita; corregirlos en las lambdas actuales es trabajo aparte.

Rutas relativas a `Lambdas_NET_LogiGho/LambdasLogiGho.Aplicacion/Logigho/`.

---

## Consistencia

| Hallazgo | Evidencia |
| --- | --- |
| Reintento x4 de cualquier POST, incluida la creación en la transportadora | `ApiLambdaCrearGuia/Infraestructura/Servicios/Generico.cs:152-237` |
| Estado `Cargado` se escribe después de crear la guía | `CG/.../CargaInterUseCase.cs:121,317` |
| `PedidosInter` sin índice único por número de guía | — |
| Insert en `PedidosInter` fallido solo se registra en log | `CG/.../CargaInterUseCase.cs:363-366` |
| Inventario lanzado en segundo plano justo antes de responder (Lambda puede congelarlo) | `CG/Function.cs:157-167` |

## Bugs

| Hallazgo | Evidencia |
| --- | --- |
| Transportadora desconocida o vacía se envía por Envia | `CG/Function.cs:130-135`, `CR/Function.cs:151` |
| D2E sin `await`: sube el texto de un `Task` como etiqueta | `CG/.../CargaD2EUseCase.cs:228` |
| Recogida D2E enviada a la API de Inter | `CG/.../CargaD2EUseCase.cs:264` |
| TCC rechazado por `/putObject` (no está en la lista) | `ApiLambdaPutObjectAOT/.../Function.cs:100` |
| Buckets `-prod` fijos en código, usados también en PreProd | `ApiLambdaCrearEtiquetaManual/.../Carga*UseCase.cs` |
| Falta de una variable TCC tumba el constructor y con él todas las transportadoras | `CargaTccUseCase.cs` (constructor) |

## Seguridad

| Hallazgo | Evidencia |
| --- | --- |
| El token Cognito del usuario se envía a Inter en cada admisión (headers `Token` y `Authorization`) | `CR/.../CargaInterUseCase.cs:174`, `CR/Infraestructura/Servicios/Generico.cs:187-194` |
| Headers con credenciales de Inter enviados a `/creacionEtiqueta` (endpoint propio) | `CR/.../CargaInterUseCase.cs:254` |
| El cliente envía `IdTienda`/`TiendaProveedor`/`Estado` sin validación de dueño | `CR/Function.cs:109-155` |
| `/pedidos`: la lambda lee primero `Token` mientras el authorizer valida `Authorization`; el JWT se decodifica sin verificar firma | `ServicioPublicoLogigho/ApiLambaConsultarPedidos/Function.cs:51-57`, `JwtClaimsHelper.cs:17-30` |
| `PUT /putObject` y `POST /actualizaInventarioLotes` sin authorizer ni API key en `7elgyten88` | AWS `get-resources` |
| Puerto 22 abierto a 0.0.0.0/0 en `sg_documentdb` | AWS `describe-security-groups` |
| Authorizer de PreProd apunta al pool Cognito de prod | AWS |

## Dinero

- Ninguna lambda reserva ni valida saldo al crear una guía. La wallet (`ResumenSaldoActualLogigho`) solo cambia al liquidar entregas y devoluciones, 2–5 días después.
- `ResumenSaldoActualLogigho` lo recalculan varios procesos con fórmulas distintas; no sirve como fuente de verdad para el cupo. Detalle en `cupo.md`.

## Infraestructura (prod)

- DocumentDB: una instancia `db.t4g.medium`, CPU máx. 99%, 495 conexiones pico, sin réplica, backup 1 día. HU: [BD: Escalamiento y endurecimiento de DocumentDB producción](https://app.clickup.com/t/86e3v5ctj).
- Lambdas en VPC en una sola subnet (`us-east-1a`) y un NAT.
- En prod, `/actualizaInventarioLotes` todavía apunta a `ApiLambdaInventarioCarga` (legacy), no a `ApiLambdaInventario`.
