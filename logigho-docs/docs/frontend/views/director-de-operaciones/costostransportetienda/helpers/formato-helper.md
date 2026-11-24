## Autor: Yadilton Torres

Fecha creacion: 2026-11-24

Estado: preprod

## Helper: formato.helper.ts

**Ubicación:** `helpers/formato.helper.ts`
**Tests:** `formato.helper.spec.ts` — funciones puras, sin Angular ni HTTP

---

## ¿Qué hace?

Cómo se muestran los datos en la tabla de costos registrados y en el modal de crear.

| Función | Ejemplo |
|---|---|
| `tipoCelda(key)` | `FleteUrbano` → `dinero` · `PorcentajeSeguro` → `porcentaje` · `Estado` → `estado` · `Tienda` → `tienda` · otro → `texto` |
| `dinero(v)` | `9520` → `$9.520` · vacío o no numérico → `—` |
| `nombreGuia(v)` | `ENTREGA` → `Entrega` · `DEVOLUCION` → `Devolución` |
| `textoTitulo(key, title)` | `EstadoGuia` se muestra como `Tipo` |
| `normalizarBusqueda(x)` | `"  GAME   Center "` → `"game center"` (espacios de más no impiden encontrar la tienda) |
