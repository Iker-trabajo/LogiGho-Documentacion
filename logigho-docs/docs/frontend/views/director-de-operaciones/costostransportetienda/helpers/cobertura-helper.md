## Autor: Yadilton Torres

Fecha creacion: 2026-11-24

Estado: preprod

## Helper: cobertura.helper.ts

**Ubicación:** `helpers/cobertura.helper.ts`
**Tests:** `cobertura.helper.spec.ts` — funciones puras, sin Angular ni HTTP

---

## ¿Qué hace?

Calcula la [Cobertura por tienda](../components/cobertura-costos.md) a partir de las colecciones `Tienda`, `CostosTransporteTienda` y `Transportadora`. Solo lectura.

---

## Constantes y tipos

| Nombre | Qué es |
|---|---|
| `TRANSPORTADORAS_REQUERIDAS` | Interrapidísimo, Servientrega, Envía y D2E (TCC no entra) |
| `ESTADOS_GUIA` | `ENTREGA`, `DEVOLUCION` |
| `TOTAL_REQUERIDO` | 8 (4 transportadoras × 2 tipos de guía) |
| `EstadoCasilla` | `activo` (existe y ACTIVO) · `inactivo` (existe pero no ACTIVO) · `falta` (no existe) |
| `NivelCobertura` | `completa` · `parcial` · `sin-costos` |

---

## Funciones

| Función | Qué hace |
|---|---|
| `normalizar(v)` | Mayúsculas, sin tildes ni espacios: `"Envía "` → `"ENVIA"` |
| `codigoPorNombre(nombre)` | Código canónico desde el nombre que venga en cualquier colección |
| `esActivo(estado)` | `true` si el estado es `ACTIVO` |
| `resolverTransportadoras(docs)` | Lee la colección `Transportadora` y devuelve Id, nombre y logo de las 4 requeridas |
| `esTarifaGenerica(idTienda)` | `true` para `IdTienda` `00000`: no es una tienda |
| `listarTarifasGenericas(costos, transportadoras)` | Lista las genéricas tal como están y marca lo que les falta |
| `calcularCobertura(tiendas, costos, transportadoras)` | Una fila por tienda con sus 8 casillas (sin genéricas). Ubica cada costo primero por `IdTransportadora` y, si no, por nombre |
| `costosDeTienda(...)` | Costos de una tienda agrupados por transportadora y tipo de guía, con sus valores (para el detalle). Si hay varios en la misma casilla, devuelve todos |
| `resumirPorTransportadora(filas)` | Números de las tarjetas: completas, solo una, sin costos |

---

## Pruebas

7 casos en `cobertura.helper.spec.ts`:

1. Reconoce nombres con tildes, espacios y mayúsculas.
2. Lee Id numérico como texto y marca D2E ausente.
3. Cruza `IdTransportadora` texto contra número y calcula faltantes.
4. Muestra tiendas que solo existen en costos y detecta duplicados.
5. Saca las tarifas genéricas `00000` de las tiendas y las lista aparte con avisos.
6. Agrupa los costos de una tienda con sus valores, aunque vengan como texto.
7. Resume completas, parciales y sin costos por transportadora.
