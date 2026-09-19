---
autor: Iker Acevedo
fecha_creacion: 2026-09-19
estado: aceptada
---

# ADR-002 — UI de gestión de accesos API: Doble lista con drag-and-drop

**Autor:** Iker Acevedo  
**Fecha:** 2026-09-19  
**Estado:** Aceptada

---

## Contexto

Antes, accesos de usuarios API se gestionaban completamente en AWS Cognito:
- Editar `custom:idTienda` y `custom:nombreTienda` en claims del JWT
- Sin UI, sin auditoría, sin control de versiones
- Cambios tardaban hasta 20s en reflejarse (refresco de token)

Con la migración a tabla MongoDB `UsuarioTienda` (ver ADR-001 backend), era necesario crear una UI para:
- Crear/editar/activar usuarios API
- Asignar/desasignar tiendas
- Auditar todos los cambios

---

## Opciones consideradas

### Opción A — Tabla simple con checkboxes

Una tabla única: usuarios en filas, tiendas en columnas.

**Pros:**
- Compacto, simple de entender

**Contras:**
- Si hay 100+ tiendas: tabla gigante
- No escala horizontalmente en mobile
- Workflow poco intuitivo (click individual en cada celda)

### Opción B — Doble lista con drag-and-drop (Elegida)

Dos listas side-by-side:
- Izquierda: "Tiendas disponibles" (no asignadas)
- Derecha: "Tiendas asignadas" (usuario actual)

Permitir:
- Drag-and-drop entre listas
- Checkboxes + botón "Asignar"
- Búsqueda en ambas listas
- Paginación en disponibles (para no saturar DOM)

**Pros:**
- Mental model claro: "mover" de un lado a otro
- Escala: con 1000 tiendas, pagina y busca
- Mobile-friendly: tabs en responsive
- Familiar: patrón estándar en admin UIs
- Drag-drop es visceral y satisfactorio

**Contras:**
- Más código (dual-list logic, paginación, búsqueda)
- Requiere `@angular/cdk/drag-drop`

### Opción C — Modal de selección múltiple

Botón "Asignar tiendas" → Modal con checkboxes de todas las tiendas.

**Pros:**
- Más compacto en pantalla principal

**Contras:**
- Modal grande con 100+ tiendas
- Imposible navegar/buscar cómodamente en modal

---

## Decisión

**Se eligió:** Opción B — **Doble lista con drag-and-drop**

**Razón:**
Escala a cualquier cantidad de tiendas. Mental model intuitivo. Permite múltiples workflows (drag, checkboxes, búsqueda). Mobile-friendly. Patrón probado en industria.

---

## Consecuencias

**Positivas:**
- ✅ Workflow fluido e intuitivo
- ✅ Escalable (paginación automática)
- ✅ Búsqueda rápida en ambas listas
- ✅ Drag-and-drop satisfactorio
- ✅ Responsive: tabs en mobile
- ✅ Sin dependencias obscuras (CoreUI + CDK es estándar Angular)

**Negativas:**
- ⚠️ Más componentes Angular involucrados
- ⚠️ Lógica de signals/computed más compleja
- ⚠️ Testing: múltiples interacciones para validar

---

## Implementación

### Estructura de estados (Signals)
- `tiendasDisponibles` (computed): filtra tiendas no asignadas
- `tiendasAsignadas` (signal): lista explícita asignada al usuario actual
- `tiendasSeleccionadasIds` (signal): tiendas chequeadas en "disponibles" (candidatas a asignar)
- `busquedaDisponibles`, `busquedaAsignadas` (signals): filtros
- `paginaDisponibles` (signal): página actual (20 por página)

### Interacciones
1. **Checkbox en disponibles** → `seleccionarTiendaCheck()` → añade/quita de `tiendasSeleccionadasIds`
2. **Botón "Asignar"** → `asignarSeleccionadas()` → mueve a `tiendasAsignadas`
3. **Drag de disponibles → asignadas** → `drop()` → añade a `tiendasAsignadas`
4. **Checkbox en asignadas** → `removerTienda()` → quita de `tiendasAsignadas`

### Paginación
- Disponibles: 20 por página (evita render de 1000+ items)
- Total páginas: `Math.ceil(disponibles.length / 20)`
- Computed `tiendasDisponiblesPaginadas` calcula slice automático

---

## Impacto en el código

| Módulo | Cambio |
| --- | --- |
| `accesos-api.component.ts` | 600+ líneas: signals, computed, métodos de gestión |
| `accesos-api.component.html` | Tabs (usuarios, tiendas, campos). Doble lista con CDK. |
| `accesos-api.component.scss` | Estilos para badges, cards, drag-drop feedback. |
| `accesos-api.repository.ts` | HTTP calls a backend CRUD. |

---

## Test plan

- [ ] Cargar usuarios API desde BD
- [ ] Seleccionar usuario y ver tiendas asignadas
- [ ] Asignar tienda via checkbox + botón
- [ ] Asignar tienda via drag-and-drop
- [ ] Remover tienda individual
- [ ] Remover todas las tiendas (con confirmación)
- [ ] Buscar en disponibles (debe paginar)
- [ ] Crear usuario nuevo
- [ ] Guardar cambios sin modificar tiendas
- [ ] Descartar cambios
- [ ] Validar Sub (Cognito UUID)
- [ ] Validar Username (alfanuméricos + punto/guion)
- [ ] Toggle `AccesoATodas` → limpia lista de tiendas
- [ ] Crear/activar/desactivar campos API
- [ ] Ver preview JSON dinámico

---

## Beneficios vs. Cognito manual

| Aspecto | Cognito manual | AccesosApiComponent |
| --- | --- | --- |
| Tiempo para asignar tienda | 5+ minutos (UI Cognito) | 30 segundos (doble lista) |
| Auditoría | En CloudTrail (opaco) | Registro limpio en MongoDB |
| Cambios inmediatos | No (esperar refresco token) | Sí (siguiente request) |
| Escalabilidad | Limitado (claims JWT) | Ilimitado (solo BD) |
| Gestión campos API | No existe | CRUD completo |

---

## Referencias

- [ADR-001 Backend: Validación de tiendas](../ApiLambdaConsultarPedidos/adr-001-migracion-validacion-tiendas.md)
- [AccesosApiComponent](accesos-api.md)
- CoreUI: https://coreui.io/angular/docs/
- Angular CDK Drag-Drop: https://material.angular.io/cdk/drag-drop/overview

---

## Historial

| Fecha | Autor | Cambio |
| --- | --- | --- |
| 2026-09-19 | Iker Acevedo | ADR creado. Doble lista con drag-and-drop adoptada para gestión UI de accesos. |
