## Autor: Yadilton Torres

Fecha creacion: 2026-11-24

Estado: preprod

## Modelos: costos-transporte.models.ts

**Ubicación:** `models/costos-transporte.models.ts`

---

| Tipo | Qué es |
|---|---|
| `Modulo` | Formulario del modal de crear. Mismos campos y tipos que usaba el componente (no se cambiaron para no tocar el envío) |
| `CampoNumero` | `{ key, titulo }` de un flete o porcentaje |
| `TipoCelda` | `'dinero' \| 'porcentaje' \| 'estado' \| 'tienda' \| 'texto'` |
| `ClaveCosto` | `IdTienda`, `IdTransportadora`, `EstadoGuia`: lo que hace único a un costo |

El ejemplo de cómo se guarda un registro está en la [vista](../costostransportetienda.md#como-se-guarda-un-registro).
