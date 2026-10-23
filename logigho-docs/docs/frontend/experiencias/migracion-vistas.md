---

## Autor: Iker Acevedo

Fecha creacion: 2026-10-17
Estado: propuesta (pendiente de aprobación)
Tipo: guía

# Migración de vistas existentes

Las vistas viejas **siguen funcionando** mientras no se migren: se muestran dentro del shell de la experiencia activa
con sus estilos actuales. Migrar una vista significa llevarla a la [estructura estándar](estructura-proyecto.md)
sin cambiar lo que ve el usuario; después se le pueden crear versiones para otras experiencias.

## Tres estados de una vista

| Estado | Qué significa | En una experiencia nueva se ve |
|---|---|---|
| **Sin migrar** | Componente original | Con sus estilos actuales, dentro del shell nuevo (solo cambian los colores que ya usen `--logigho-*` o tokens) |
| **Migrada** | Estructura estándar con versión `clasica` | Versión `clasica` con los tokens de la experiencia (respaldo) |
| **Con versión propia** | Tiene `experiencias/<id>/` | Su versión diseñada para esa experiencia |

## Pasos para migrar (un PR por paso)

Antes de empezar, tomar pantallazos de los estados principales (cargando, con datos, vacía, con filtros, con error).

1. **Helpers:** mover las reglas puras y escribir sus pruebas.
2. **Repository:** mover el acceso a datos.
3. **Store + facade:** mover estado y acciones; el componente queda como versión visual que solo usa el facade.
4. **Página + registro:** crear `<vista>.page.ts` y `<vista>.experiencias.ts`; mover el componente a
   `experiencias/clasica/`; la ruta apunta a la página con `data.vista`.
5. **Tokens:** reemplazar colores fijos por tokens en la versión `clasica`.

Después de cada paso: compilar, correr pruebas y comparar con los pantallazos.

## Cómo clasificar cada método del componente viejo

| Señal | Destino |
|---|---|
| Usa `HttpClient`, `ConsumoGenericoService`, descompresión o workers | `repository/` |
| Recibe datos y devuelve datos, sin efectos | `helpers/` |
| Listas, filtros, selección, cargando, error | `state/*.store.ts` |
| Llama al API y actualiza el estado; decide permisos | `state/*.facade.ts` |
| Modal abierto, dropdown abierto, pestaña visible | La versión visual |

## Orden de migración

1. **El shell** (`default-layout`) primero: es la base de todas las experiencias.
2. **`tienda/pedidos`**: la vista piloto ([ejemplo completo](ejemplo-tienda-pedidos.md)).
3. Después, por uso y por impacto. El orden lo define Iker y se registra en la tabla de cobertura.

## Cobertura

Se actualiza en cada PR que migre una vista o agregue una versión.

| Vista | Clave | Estado | `clasica` | Experiencia B | Experiencia C |
|---|---|---|---|---|---|
| Shell | — | Sin migrar | — | — | — |
| Pedidos | `tienda.pedidos` | Sin migrar | — | — | — |

## Errores comunes

| Error | Corrección |
|---|---|
| Migrar y cambiar comportamiento en el mismo PR | Primero migrar sin cambios; el cambio funcional va después |
| La versión visual inyecta el repository "para una sola cosa" | Esa cosa es una acción del facade |
| Proveer el facade en `root` | Se provee en la página |
| Dejar lógica en la versión `clasica` "porque ya estaba ahí" | Toda la lógica sube al cerebro antes del paso 4 |

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-10-17 | Iker Acevedo | Creación de la propuesta |
