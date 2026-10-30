# Configuración de Entregas — panel de administración

Panel donde quien administra la métrica **configura el cálculo, lanza y revisa corridas y define el ajuste visual** (historia 016 y spec 005). Lo que ven los usuarios está en [Entregas](metricas-efectividad-ciudad.md).

| | |
|---|---|
| **Ruta** | `/app/administracion/configuracion-entregas` |
| **Módulo de menú** | `Modulo` ID `170` — `Configuracion Entregas \| Administracion` (padre `7`), icono `cilSettings`; roles `CEO` y `Desarrollador` |
| **Código** | `SitioLogiGho/src/app/views/administracion/configuracion-entregas/` |
| **Backend** | `GET/PUT .../configuracion`, `GET/POST .../corridas`, `GET .../corridas/{id}` ([API](../../api/metricas.md)) |

!!! info "El panel no es la barrera"
    El menú oculta la opción a quien no tiene rol, pero **el backend vuelve a validar el rol** en cada llamada (403 si no es `CEO` ni `Desarrollador`).

El encabezado azul muestra la versión de la configuración, el alcance de la corrida programada y el mínimo de guías, con los atajos **Recalcular datos** y **Ver corridas**. Arriba a la derecha, un indicador "Sin corrida en curso / Corrida en curso…".

## Pestaña Parámetros

Todos los cambios aplican **desde la próxima corrida**; no reescriben lo ya calculado. Se guardan con **Guardar parámetros** (un PUT validado por el backend) o se descartan con **Restablecer**.

- **Alcance de la corrida programada:** todo el histórico, últimos 6 o últimos 3 meses. La corrida del domingo 8:00 a. m. lo usa.
- **Muestra mínima de guías por transportadora** (− / +, de 5 en 5, entre 1 y 200). El botón **?** explica qué es: la cantidad mínima de guías con resultado final para que un porcentaje sea confiable; por debajo, el porcentaje se ve en gris como "sin muestra" y no cuenta en la mejor opción ni en los rankings. Trae un ejemplo (con 30, 8 de 10 guías entregadas dan 80 % pero no se recomienda).
- **Transportadoras que cuentan:** informativo (sale del mapa de la configuración). Las excluidas (XCARGO) se listan aparte y TCC **no se muestra** hasta que esté en la base.
- **Clasificación de estados.** La lista sale de la **auditoría de la última corrida finalizada** (`estadosObservados`, sumando transportadoras) más los estados ya configurados:
  - *Estados que cuentan como "entregado"* y *como "devuelto"*: cada chip muestra su cantidad de guías y, al pasar el cursor, qué transportadoras lo reportan. Un estado **no puede estar en las dos listas**: al estar en una, el chip de la otra se bloquea.
  - *Estados que no cuentan* (tránsito, anuladas…): solo informativo. Para que cuente, se marca arriba.
  - *Estados que se resuelven por fecha* (**Archivada**): **no se escoge**; se resuelve sola (solo fecha de entrega → entrega; solo fecha de devolución → devolución; ambas o ninguna → no cuenta).
- **Estado de la configuración:** versión, quién la actualizó y fecha legible.

## Pestaña Corridas

Dos tarjetas separadas:

1. **Lanzar una corrida:** alcance de la corrida manual y botón **Lanzar corrida** (con confirmación). Solo puede haber una a la vez (409 si hay otra viva).
2. **Historial de corridas:** las últimas 20, con inicio en texto legible hora de Bogotá (**"29 de oct de 2026 a las 11:54 a. m."**), origen, alcance, estado, guías leídas y duración (**"1 min 53 s"**). Se refresca cada 10 s mientras hay una en curso.
   - Una corrida `EN_CURSO` de **más de 20 minutos** se muestra como **CAÍDA** y no bloquea el botón de lanzar.
   - **Ver detalle** abre la **auditoría** de esa corrida: entregas, devoluciones, % de ciudad reconocida, quién la solicitó, cuándo terminó, las guías que no cuentan **por motivo** (traducido) y las **ciudades que no se reconocieron** (hasta 50, con departamento y guías). Si la corrida falló, muestra el error.

## Pestaña Ajuste de porcentajes

Define cómo se elevan los porcentajes que ven los usuarios (regla y parámetros en [Ajuste visual](../../backend/metricas/ajuste-visual.md)).

- **Estado del ajuste:** *Apagado (porcentaje real)* o *Activo*.
- **Porcentaje mínimo a mostrar** (1 a 99) y **modo** (*Por ciudad* / *Global*).
- **Guardar ajuste** (el mismo PUT de la configuración) y **Restablecer**. Los usuarios lo ven al guardar y recargar Entregas.
- **Vista previa real → mostrado:** tabla por ciudad y transportadora con el porcentaje real tachado y el mostrado en negrita, buscador de ciudad o departamento y "Ver más". Incluye la cantidad de celdas con muestra y los promedios real y mostrado. La vista previa usa los valores **sin guardar** y funciona aunque el ajuste esté apagado.

## Qué se retiró

El boceto original traía las pestañas **Comparar**, **Vista del usuario** e **Historial** (borrador, publicación versionada, restaurar). El backend de esta historia no lo soporta, así que **se quitaron** del panel (y sus endpoints y estilos). Si se retoma, necesita colecciones y endpoints nuevos (backlog M-132 a M-140).

## Cómo está hecho

- `ConfiguracionEntregaService` (`services/`): solo los endpoints que existen; adapta PascalCase/camelCase.
- `utils/configuracion.utils.ts` (con tests): clasificación de estados (`construirEstados`, `asignarClase`: Archivada no se asigna), fechas y duraciones legibles hora Bogotá, motivos en español, transportadoras visibles y `corridaVigente` (RN-08: una `EN_CURSO` de más de 20 min está caída).
- Correcciones del 2026-10-29: el id de la corrida llegaba vacío (el adaptador leía `Id` en vez de `CorridaId`) y el % de ciudad resuelta no se leía (el campo es `PorcentajeCiudadResuelta`).
