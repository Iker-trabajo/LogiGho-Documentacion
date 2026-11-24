---
autor: Yadilton Torres
fecha_creacion: 2026-11-24
ultima_actualizacion: 2026-11-24
estado: desarrollo
nivel: 2
---

# Logi: seguridad por rol

**Autor:** Yadilton Torres

**Ubicación:** `SitioLogiGho/src/app/components/logi-pet/logi-mascot.component.ts`

---

## Problema

Logi solo navegaba a pantallas del menú del usuario, pero **mostraba información de módulos que el rol no tenía**:

- «Pregúntale a Logi» y las preguntas frecuentes mostraban guías de otros módulos (ej. un rol Tienda podía leer los pasos de Controller).
- Los consejos, «¿Qué hago aquí?» y «Muéstrame cómo» explicaban cualquier pantalla abierta a mano, aunque el rol no la tuviera.

Además, el control de rutas del layout tenía un hueco: un **módulo padre abría todas sus pantallas hijas**. Ver [Acceso a módulos](../../core/acceso-modulos.md).

---

## Regla

Logi pregunta a `AccesoModulosService.puede(ruta)`, que usa la **misma regla que el layout**. Mientras no carguen los módulos responde `false`: Logi no habla de lo que no sabe si el usuario puede ver.

| Parte de Logi | Qué hace ahora |
|---|---|
| Buscador (respuestas) | Solo guías sin `ruta` o cuya `ruta` el rol puede abrir |
| Preguntas frecuentes | Las de la pantalla, si el rol puede estar en ella. Si no, las generales (`desembolso`, `crear-pedido`, `guia-no-sale`) que el rol tenga. Si no tiene ninguna, las que sirven para todos (`no-veo-modulo`, `soporte`, `atajos`) |
| Abrir una guía | `abrirGuia()` no abre una guía de un módulo ajeno, aunque se la pidan directo |
| «Llévame» y buscador de pantallas | Solo URLs del menú del usuario (ya era así) |
| Consejo al entrar | No da consejos de pantallas sin permiso |
| «¿Qué hago aquí?» | En una pantalla sin permiso dice «No tienes acceso a esta pantalla con tu rol. Si la necesitas, pídele permiso a un administrador.», sin consejos ni preguntas |
| «Muéstrame cómo» | No hace el recorrido en pantallas sin permiso |

---

## Métodos

```ts
/** Una guía solo se muestra si su pantalla es de un módulo que el rol puede abrir */
private guiaVisible(guia: GuiaLogi): boolean {
  return !guia.ruta || this.acceso.puede(guia.ruta);
}

/** Si el usuario puede estar en la pantalla actual */
private pantallaPermitida(): boolean {
  return this.acceso.puede(this.router.url);
}
```

---

## Corrección de datos

La guía de **Ingreso Devoluciones** apuntaba a `/logistica/ingreso-devoluciones`, ruta que no está en la colección `module`; con la regla nueva no le habría salido a nadie. Ahora apunta a `/logistica/devolucion`.

---

## Pruebas

`logi-mascot.component.spec.ts` (sin pintar el componente):

- El buscador no responde con guías de módulos ajenos y sí con las propias.
- Mientras no carguen los módulos solo salen guías que no dependen de un módulo.
- Las preguntas frecuentes excluyen las de módulos ajenos; sin ninguna, salen las que sirven para todos.
- «¿Qué hago aquí?» en una pantalla sin permiso dice «No tienes acceso»; en una propia la explica.
- No abre una guía ajena aunque se la pidan directo.

---

## Observaciones

- Esto cierra lo que **se ve**. El backend debe seguir validando el rol en cada servicio.
- Al agregar una guía nueva, su `ruta` debe ser la misma que tiene el módulo en la colección `module` (sin `/app`); si no, no le saldrá a nadie.
