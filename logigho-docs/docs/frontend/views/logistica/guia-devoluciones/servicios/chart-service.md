---

## Autor: Adalberto González
Fecha creacion: 2026-06-03  
Estado: produccion

# Servicio: ChartComputerService

**Ubicación:** `src/app/views/logistica/guias-devoluciones/services/chart-computer.service.ts`  
**Scope:** `providedIn: 'root'` Puede ser utilizado en el resto de el proyecto de el front

---

## ¿Qué hace?

Este servicio se encarga de preparar la información para que pueda ser mostrada en la interfaz de usuario. 
A partir de los registros de devoluciones, genera tanto los datos necesarios para el gráfico principal como la estructura utilizada por la tabla resumen.

---

## Métodos

### `computar(datos: DevolucionRow[]): ChartResult`

Punto de entrada principal. Aplica los filtros activos via `FilterService`, luego calcula chart y tabla en una sola pasada.

| Parámetro | Tipo | Descripción |
|---|---|---|
| `datos` | `DevolucionRow[]` | Registros completos en memoria, sin filtrar |

**Retorna:** `ChartResult` — objeto con todas las propiedades listas para asignar al template.

---

## Interfaz `ChartResult`

| Propiedad | Tipo | Descripción |
|---|---|---|
| `chartData` | `DayPoint[]` | Barras del chart ordenadas por fecha |
| `chartMax` | `number` | Valor máximo del eje Y, redondeado a múltiplos de 300 |
| `yTicks` | `string[]` | 5 etiquetas del eje Y: máximo, 75%, 50%, 25%, 0 |
| `xTicks` | `Set<string>` | Labels que deben mostrarse en el eje X (domingos + hoy) |
| `chartScrollWidth` | `number \| null` | Ancho total del chart en px para el scroll horizontal |
| `tablaResumen` | `TablaFila[]` | Árbol jerárquico: mes → fecha → tienda → guía |
| `labelMes` | `string` | Texto del subtítulo: `Junio 2026`, `Últimos 2 meses`, etc. |
| `tablaDetalle` | `DevolucionRow[]` | Registros del mes activo para la tabla de detalle |
| `totalValorDeclarado` | `number` | Suma de `ValorDeclarado` de `tablaDetalle` |

---

## Flujo interno

```
computar(datos)
  → filterService.aplicar(datos)       → vista filtrada
  → filterService.getPrefixesMesActivo() → prefijos YYYY-MM del mes activo
  → calcular(vista, prefixes, ...)
      → recorre vista UNA SOLA VEZ:
          - acumula DayPoint en mapaChart (Map<dateKey, DayPoint>)
          - acumula TablaFila[] en porFecha (Map<fecha, Map<tienda, guías>>)
      → finalizarChart()   → chartData, chartMax, yTicks, xTicks, chartScrollWidth
      → finalizarTabla()   → tablaResumen, tablaDetalle, totalValorDeclarado
  → retorna ChartResult
```

---

## Endpoints que consume

Ninguno. Este servicio no hace HTTP.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-06-03 | Adalberto González | Se creo el servicio |

---

## Observaciones