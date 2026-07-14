## Autor: Iker Acevedo
Fecha creacion: 2026-07-13

Estado: produccion

## Lambda: ApiLambdaObtenerCuentasPrincipales

**Accionador:** Step Functions (`Task`)

**AOT:** No

**Posición en el pipeline:** 2 de 5

---

## ¿Qué hace?

Devuelve la lista de **cuentas madre activas** de Pancake, cada una con su token de acceso. Lee la colección `PancakeCuentasPrincipales`, filtra por `estado: "Activo"` y proyecta solo lo necesario para el primer `Map`: el id de la cuenta y su token.

Cada cuenta madre puede tener muchas páginas asociadas; esta lambda es el punto de partida para recorrerlas.

---

## Request

No recibe datos propios. En el Step Function se ejecuta con el estado que traiga el pipeline (su salida se guarda en `$.cuentas`).

```json
{}
```

---

## Response

Un **array** de cuentas activas:

```json
[
  { "cuenta_id": "10001", "token_acceso": "eyJhbGciOiJ..." },
  { "cuenta_id": "10002", "token_acceso": "eyJhbGciOiJ..." }
]
```

| Campo | Tipo | Descripción |
| ----- | ---- | ----------- |
| `cuenta_id` | `string` | Identificador de la cuenta madre |
| `token_acceso` | `string` | Token de la cuenta (se usa en la lambda 3 para listar sus páginas) |

> En el Step Function esta salida se guarda en `$.cuentas` y alimenta el **1er `Map`** (una iteración por cuenta madre).

---

## Flujo interno

```
FunctionHandler (Function.cs)
  -> ObtenerCuentasPrincipalesUseCase.EjecutarAsync()
       -> ICuentasRepository.ObtenerActivasAsync()
            -> MongoDB: PancakeCuentasPrincipales
               Filter.Eq(estado, "Activo")
               Proyección -> { cuenta_id, token_acceso }
  -> [RESULT] IReadOnlyList<CuentaPrincipal>
```

---

## Arquitectura Clean Architecture

```
ApiLambdaObtenerCuentasPrincipales/
├── Function.cs
├── Dominio/
│   └── Entidades/
│       └── CuentaPrincipal.cs           ← { cuenta_id, token_acceso }
├── Aplicacion/
│   ├── CasosUso/
│   │   └── ObtenerCuentasPrincipalesUseCase.cs
│   └── Interfaces/
│       └── Repositorios/
│           └── ICuentasRepository.cs    ← Contrato del repositorio
└── Infraestructura/
    └── Repositorio/
        └── DocumentRepository.cs        ← Acceso a MongoDB
```

---

## Variables de entorno

| Variable | Descripción | Valor ejemplo |
| -------- | ----------- | ------------- |
| `CADENA_CONEXION` | Cadena de conexión MongoDB (cifrada AES-256-ECB) | String cifrado |
| `DATABASE_NAME` | Nombre de la base de datos MongoDB | `"LogighoDB"` |

---

## Configuración Lambda

| Parámetro | Valor |
| --------- | ----- |
| Runtime | `dotnet8` |
| Handler | `ApiLambdaObtenerCuentasPrincipales::ApiLambdaObtenerCuentasPrincipales.Function::FunctionHandler` |
| Memory | `256 MB` |
| Timeout | `15 segundos` |
| Architecture | `x86_64` |

---

## Colección `PancakeCuentasPrincipales`

```json
{
  "_id": "10001",
  "nombre": "Cuenta Principal Norte",
  "tokenAcceso": "eyJhbGciOiJ...",
  "estado": "Activo",
  "fechaCreacion": "2026-07-01",
  "fechaActualizacion": "2026-07-10"
}
```

Solo las cuentas con `estado: "Activo"` entran al pipeline. Para "apagar" una cuenta basta con cambiar su `estado` en MongoDB — sin redesplegar.

---

## Historial de cambios

| Fecha | Autor | Cambio |
| ----- | ----- | ------ |
| 2026-07-13 | Iker Acevedo | Creación. Lectura de cuentas madre activas desde `PancakeCuentasPrincipales`. |

---

## Observaciones

- El token viaja en texto (tal como está en la colección); el consumo posterior (lambda 3) lo pone en la query de Pancake, nunca se loguea.
- Apagar/prender una cuenta es una operación de datos (`estado`), no de código.
