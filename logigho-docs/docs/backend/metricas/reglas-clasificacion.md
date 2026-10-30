# Métricas — Reglas de clasificación

Todas las reglas comparan el texto **normalizado**: mayúsculas, sin tildes, espacios colapsados, sin espacios al borde. Las listas de estados, el mapa de transportadoras, las excluidas y los alias de ciudad se editan desde el front **sin desplegar código**.

## Clasificación de cada guía (RN-01)

Cada guía cae en **una** clase:

| Clase | Regla inicial (configurable) |
|---|---|
| **ENTREGA** | `Estado` ∈ `EstadosEntrega` = {`PAGADA`, `ENTREGADA`} |
| **DEVOLUCION** | `Estado` ∈ `EstadosDevolucion` = {`DEVOLUCION COMPLETADA`, `DEVOLUCION RATIFICADA`, `DEVOLUCION`} |
| **POR FECHA** | `Estado` ∈ `EstadosPorFecha` = {`ARCHIVADA`}: con solo `Fecha Entrega` → ENTREGA; con solo `Fecha Devolucion` → DEVOLUCION; con ambas o ninguna → NO_CUENTA (`POR_FECHA_INDEFINIDA`) |
| **NO_CUENTA** | Todo lo demás: tránsito, `ANULADA`, `CREADO`, `DEVOLUCION REGIONAL` (transitorio), `INDEMNIZADA`, `SINIESTRADO`… |

"Fecha presente" = el campo existe, no es nulo y no es vacío tras recortar. No se interpreta el formato. Un estado no puede estar en dos listas: el PUT de configuración lo rechaza (400).

## Porcentaje (RN-02)

`% entrega = ENTREGA / (ENTREGA + DEVOLUCION)`. NO_CUENTA no entra ni al numerador ni al denominador. Si `ENTREGA + DEVOLUCION = 0`, el % es `null` (se muestra "—").

## Mínimo de muestra (RN-03)

`Finales = ENTREGA + DEVOLUCION`. Si `Finales < MinimoGuias` (por defecto **30**), la celda se marca "sin muestra" y **no cuenta** para la mejor opción ni para los rankings, aunque el % se sigue mostrando. El mínimo es el de la configuración (`MinimoGuiasPorDefecto`) y lo cambia el administrador en el panel; el filtro por usuario que se pensó al inicio **se retiró** (decisión de Iker, 2026-10-29).

## Transportadora (RN-04)

Se normaliza el texto y se aplica `MapaTransportadoras` (texto normalizado → código canónico):

| Texto en los datos | Código canónico |
|---|---|
| `INTERRAPIDISIMO`, `INTER`, `INTERRAPIDISO`, `INTERAPIDISIMO` | `INTERRAPIDISIMO` |
| `ENVIA`, `ENIVA`, `ENVIO` | `ENVIA` |
| `SERVIENTREGA` | `SERVIENTREGA` |
| `D2E` | `D2E` |
| `TCC` | `TCC` (el front lo **oculta** hasta que sus guías estén en `PedidosInter`) |
| `D2` | `XCARGO` |

- `TransportadorasExcluidas` = {`XCARGO`} → NO_CUENTA motivo `TRANSPORTADORA_EXCLUIDA`.
- Texto sin mapeo, nulo o vacío → NO_CUENTA motivo `TRANSPORTADORA_DESCONOCIDA` (aparece en la auditoría para que el administrador lo mapee).

## Ciudad (RN-05)

Se resuelve a la ciudad canónica de `Ciudades` (clave `dane.mun8`). Se intentan estos pasos en orden y se detiene en el primero que da **un único** resultado:

1. `Ciudad` vacía y `Departamento` con `/` → ciudad = izquierda, departamento = derecha (caso Servientrega: `BOGOTA/CUNDINAMARCA`).
2. `Departamento` con `/` → departamento = parte derecha (`MEDELLIN/ANTIOQUIA`).
3. Alias extra de la configuración (`AliasCiudades`).
4. Coincidencia exacta nombre + departamento.
5. Si la ciudad termina con el nombre del departamento (`CARTAGENA BOLIVAR`), quitarlo y repetir el paso 4.
6. Coincidencia solo por nombre si ese nombre es único en todo el catálogo.
7. Nada único → NO_CUENTA motivo `SIN_CIUDAD`.

**Meta:** ≥ 95 % de las guías finales con ciudad resuelta; el resto sale en la auditoría.

## Mes (RN-06)

Mes de la guía = primeros 7 caracteres de `FechaCarga` (`yyyy-MM`; el formato observado es `yyyy-MM-dd HH:mm:ss`). Vacío u otro formato → NO_CUENTA motivo `SIN_FECHA_CARGA`.

## Alcance de la corrida (RN-07)

- `TODO`: todos los meses.
- `ULTIMOS_N` (6 o 3): desde el día 1 del mes que está N-1 meses antes del actual (hora Bogotá) hasta hoy.
- Los meses fuera del alcance no se tocan.

## Una corrida a la vez (RN-08)

Candado atómico. Corrida `EN_CURSO` de menos de 20 min bloquea una nueva (manual 409, programada `OMITIDA`); de más de 20 min se libera por caída.

## Permisos (RN-09)

- **Ver** el resumen: cualquier usuario autenticado.
- **Administrar** (lanzar corrida, editar config, ver corridas): roles `CEO` y `Desarrollador`.
- El backend primero **verifica la firma del token** y luego busca el `sub` en `Users` ([Seguridad](seguridad.md)). El front solo oculta; no es la barrera.

## Estados que quedan fuera por decisión pendiente

`Facturado` (Inter, 11 guías), `INDEMNIZADA` (Envía, 11) y `EN PROCESO DE INDEMNIZACION` (Envía, 9) siguen como NO_CUENTA hasta que Iker las revise (decisión 2026-10-27). No están en ninguna lista de la configuración.
