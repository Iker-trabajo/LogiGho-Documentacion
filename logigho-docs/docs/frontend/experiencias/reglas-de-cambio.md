---

## Autor: Iker Acevedo

Fecha creacion: 2026-10-17
Estado: propuesta (pendiente de aprobación)
Tipo: reglas de comportamiento

# Reglas de cambio entre experiencias

Estas reglas definen cómo se comporta la plataforma cuando el usuario elige, cambia o pierde una experiencia.
Son iguales para todas las experiencias presentes y futuras.

## R1. Una experiencia por usuario, aplicada a toda la plataforma

El usuario elige **una** experiencia y se aplica a todo: shell, menú, estilos y vistas. No se mezclan
experiencias entre vistas.

## R2. Lo visual cambia; nada más

Al cambiar de experiencia **no cambian** las URLs, los permisos, los datos, los filtros guardados en el backend ni las
acciones disponibles. Un enlace copiado en una experiencia abre la misma pantalla en otra.

## R3. Respaldo con `clasica`

Si la vista actual no tiene versión para la experiencia activa, se monta su versión `clasica` con los tokens de la
experiencia activa. El usuario no ve errores ni avisos técnicos. El equipo sigue el avance en la
[tabla de cobertura](migracion-vistas.md#cobertura).

## R4. Experiencia inicial

1. Si el usuario tiene una experiencia guardada y disponible para él, se usa esa.
2. Si no, se usa `clasica`.

Mientras se consulta la preferencia en el servidor, se usa la última experiencia guardada en el dispositivo
(solo el id, sin datos sensibles) para no mostrar un parpadeo. Si el servidor responde algo distinto, gana el
servidor.

## R5. Cómo se cambia

1. El usuario abre el selector de experiencia (presente en todos los shells).
2. Ve solo las experiencias disponibles para él (ver R7).
3. Si la vista actual tiene cambios sin guardar, se le pide confirmación antes de cambiar.
4. Se guarda la preferencia en el servidor.
5. Se reemplaza el shell y se vuelve a montar la vista actual en la misma URL, **sin recargar el navegador**.
   La vista vuelve a consultar sus datos (o los toma del caché del repository si lo tiene).

Si guardar en el servidor falla, la experiencia no cambia y se muestra un aviso. Nunca se queda la interfaz en
una experiencia que el servidor no tiene registrada.

## R6. Si la experiencia guardada ya no existe

Si el id guardado no existe en el registro o está `retirada`, se usa `clasica` y se actualiza la preferencia en el
servidor.

## R7. Disponibilidad por estado

| Estado | Disponible para |
|---|---|
| `borrador` | Rol Desarrollador |
| `beta` | Usuarios habilitados para pruebas |
| `general` | Todos |
| `retirada` | Nadie |

La disponibilidad la decide el **servidor** al devolver las preferencias, no el frontend.

## R8. Tema claro / oscuro

Una experiencia puede ofrecer modo claro y oscuro. Es una opción **dentro** de la experiencia, no una experiencia
aparte, y se guarda en la misma preferencia.

## R9. Una vista no conoce la experiencia activa

El cerebro de una vista (facade, store, repository, helpers) nunca pregunta qué experiencia está activa. Si una
regla de negocio depende de la experiencia, está mal ubicada.

## R10. Nada se carga sin usarse

El shell, el UI kit y las versiones de vistas de una experiencia se descargan solo cuando esa experiencia está
activa (carga diferida).

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-10-17 | Iker Acevedo | Creación de la propuesta |
