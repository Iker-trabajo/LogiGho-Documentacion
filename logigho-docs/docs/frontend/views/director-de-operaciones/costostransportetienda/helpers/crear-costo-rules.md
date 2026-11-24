## Autor: Yadilton Torres

Fecha creacion: 2026-11-24

Estado: preprod

## Helper: crear-costo.rules.ts

**Ubicación:** `helpers/crear-costo.rules.ts`
**Tests:** `crear-costo.rules.spec.ts` — funciones puras, sin Angular ni HTTP

---

## ¿Qué hace?

Reglas del modal **Crear costo transporte tienda**. Solo revisan antes de crear: si todo está bien, el componente llama a `onCreateOrModify()` como siempre. **No cambian lo que se envía.**

Salen de cómo se guardan los costos hoy:

- **Fletes:** pesos enteros, sin puntos ni comas (`9520`, `11200`, `36750`).
- **Porcentajes:** número con punto decimal y máximo 2 decimales (`4`, `2.5`, `35`).

---

## Constantes

| Nombre | Valor |
|---|---|
| `CAMPOS_FLETE` | Urbano, Regional, Nacional, Reexpedido, Nacional metropolitano, Nacional municipal |
| `CAMPOS_PORCENTAJE` | Kilo adicional, Recaudo, Seguro |
| `FLETE_MIN` | 1.000 |
| `FLETE_MAX` | 1.000.000 |

---

## Funciones

| Función | Qué hace |
|---|---|
| `esPorcentaje(key)` | `true` si el campo empieza por `Porcentaje` |
| `textoValor(v)` | Valor como texto sin espacios al inicio ni al final; `null`/`undefined` → `''` |
| `errorCampo(key, valor, intento)` | Mensaje de error o `null` (tabla abajo). Un campo vacío solo es error si ya se intentó crear |
| `teclaProhibida(key, tecla)` | Fletes: bloquea `e E + - . ,` y espacio. Porcentajes: igual pero deja el punto |
| `limpiarPegado(key, texto)` | Al pegar: quita espacios (también los de en medio), `$` y `%`. Fletes: quita separadores de miles (`"$ 9.520"` → `9520`). Porcentajes: coma a punto (`" 2,5 %"` → `2.5`). Si no queda número, devuelve el texto para que la validación avise |
| `esDuplicado(registros, idTienda, idTransportadora, estadoGuia)` | Mismo criterio que `checkForDuplicateTienda`, comparando sin espacios de más (`" 157174"` = `"157174"`) |
| `problemasCrear(datos)` | Lista de cosas por corregir (vacía = se puede crear): tienda, transportadora, tipo de guía, estado, valores que faltan, valores mal escritos con su campo y duplicado |

### Mensajes de `errorCampo`

| Caso | Fletes | Porcentajes |
|---|---|---|
| Vacío (después de intentar crear) | Falta este dato. | Falta este dato. |
| No es número | Solo números. | Solo números. |
| Negativo | No puede ser negativo. | No puede ser negativo. |
| 0 | Debe ser mayor que 0. | Debe ser mayor que 0. |
| Decimales | Pesos enteros, sin puntos ni comas (ej. 9520). | Más de 2 → Máximo 2 decimales, con punto (ej. 2.5). |
| Fuera de rango | < 1.000 → Parece muy bajo · > 1.000.000 → Parece muy alto | > 100 → Máximo 100%. |

---

## Ejemplo

```ts
problemasCrear({
  idTienda: '157174', idTransportadora: '10002', estadoGuia: 'DEVOLUCION', estado: 'ACTIVO',
  valores: { FleteUrbano: '', PorcentajeSeguro: 150, /* … */ },
  duplicado: false,
});
// ['Faltan 1 valor: urbano.', '% seguro: Máximo 100%.']
```
