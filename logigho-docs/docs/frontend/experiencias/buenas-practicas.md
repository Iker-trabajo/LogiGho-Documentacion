---

## Autor: Iker Acevedo

Fecha creacion: 2026-10-17
Estado: propuesta (pendiente de aprobación)
Tipo: guía rápida

# Buenas prácticas: sí y no

Lista corta para revisar un PR en minutos. El detalle está en cada documento del estándar.

## Sí

| Hacer | Por qué |
|---|---|
| Escribir primero la lógica (models, repository, helpers, state) y después lo visual | La lógica no depende de ninguna experiencia |
| Poner cada regla de negocio en `helpers/` con su prueba | Se reutiliza en todas las experiencias y se prueba sin Angular |
| Proveer store y facade en la página (`providers`) | El estado vive y muere con la vista |
| Usar signals y `computed` para el estado derivado | Se calcula una vez por cambio, no en cada render |
| Componentes standalone con `ChangeDetectionStrategy.OnPush` | Rendimiento y comportamiento predecible |
| `input()` y `output()` en componentes del UI kit | Componentes reutilizables y fáciles de probar |
| Usar tokens semánticos en todo el SCSS | Una experiencia nueva se ve bien sin tocar el componente |
| Mantener `clasica` en toda vista | Es el respaldo de todas las experiencias |
| PRs pequeños que se puedan desplegar solos | Menos riesgo, revisión más rápida |
| Documentar la vista con la plantilla al terminar | La documentación es parte de "terminado" |

## No

| Evitar | Hacer en su lugar |
|---|---|
| Copiar lógica entre versiones de una vista | Subirla al facade o a `helpers/` |
| `if (experiencia === 'x')` en cualquier parte | Una versión por experiencia en su carpeta |
| Llamar al API desde una versión visual o un shell | Acción en el facade |
| Decidir permisos con `roles.includes(...)` en una versión visual | `facade.permisos()` |
| Usar `metodoGenerico` en código nuevo | Endpoint específico y seguro |
| Colores fijos, `!important`, `::ng-deep` nuevo | Tokens y tokens de componente |
| Importar de otra experiencia | Componente neutro en `components/` |
| Crear abstracciones "por si acaso" | Esperar al segundo caso real |
| Renombrar ids de experiencias o claves de vistas | Crear uno nuevo y retirar el anterior |
| Migrar una vista y cambiar su comportamiento en el mismo PR | Primero migrar sin cambios; después el cambio funcional |

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-10-17 | Iker Acevedo | Creación de la propuesta |
