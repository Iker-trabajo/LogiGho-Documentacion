# Crear un módulo del menú por base de datos

El menú lateral del SitioLogiGho **no** se arma desde `_nav.ts`, sino desde la
colección **`Modulo`** de Mongo (`LogighoDB`). `default-layout.component.ts`
(`fetchModules` → `buildNavItems`) la lee, filtra por `roles` del usuario y
construye el árbol. **Agregar una opción al menú = insertar un documento en esa
colección.**

## Campos del documento `Modulo`

| Campo | Tipo | Para qué sirve |
|-------|------|----------------|
| `ID` | string numérica | Id único. El menú se ordena por `Number(ID)` ascendente. |
| `Nombre` | string | `"Etiqueta \| NombrePadre"`. Antes del `\|` es la etiqueta; después, el grupo padre. Sin `\|` = primer nivel. |
| `ruta` | string | Ruta Angular con `/app` (ej. `/app/tienda/entregas`). **Minúscula** `ruta`: es la que lee el código. |
| `moduloPadre` | string \| null | `ID` del módulo padre; `null`/`""`/`"No Aplica"` para primer nivel. |
| `roles` | array string | Perfiles que ven el módulo. `["Todos"]` = todos. Sin array válido no aparece. |
| `iconComponent` | string | Icono CoreUI (`cilTruck`, `cilSettings`…). Los submenús heredan el del padre. |
| `estado` | string | `"ACTIVO"` para aparecer (el front pide `module?estado=ACTIVO`). Con **minúscula**, como en los documentos observados. |

## Reglas que impone el código

1. El orden dentro de un grupo lo da el `ID` numérico (no hay campo "orden").
2. El padre se enlaza por `moduloPadre` (ID) **y** por la parte tras el `|` de
   `Nombre` (nombre del padre): mantener ambos consistentes.
3. `roles` siempre array; `["Todos"]` o la lista exacta de perfiles.
4. `estado: "ACTIVO"` o no entra al menú.
5. La ruta Angular (`routes.ts`) y el componente deben existir, o el menú lleva
   a un 404 / redirect al inicio.

## Plantilla del JSON

```json
{
  "ID": "<siguiente_id_libre>",
  "Nombre": "<Etiqueta> | <NombrePadre>",
  "ruta": "/app/<padre>/<slug>",
  "moduloPadre": "<ID_del_padre>",
  "roles": ["<Rol1>", "<Rol2>"],
  "iconComponent": "<cilIcono>",
  "estado": "ACTIVO"
}
```

Primer nivel (sin padre): `moduloPadre: null`, `Nombre` sin `|`.

## Ejemplos reales

Los dos módulos de Entregas, tal como están en `Modulo` de PreProd (observado 2026-10-30). En producción se insertaron los mismos (verificados sus `ID` y `Nombre`):

```json
{ "ID": "169", "Nombre": "Entregas | Mi Tienda",
  "ruta": "/app/tienda/entregas", "moduloPadre": "1", "estado": "ACTIVO",
  "iconComponent": "cilTruck",
  "roles": ["Administrador", "Tienda", "Controller", "CEO", "Dropshipper", "COO",
            "Logistica", "Accionista", "GLOBAL ACCOUNT", "TiendaNW", "BPO"] }

{ "ID": "170", "Nombre": "Configuracion Entregas | Administracion",
  "ruta": "/app/administracion/configuracion-entregas", "moduloPadre": "7",
  "estado": "ACTIVO", "iconComponent": "cilSettings",
  "roles": ["CEO", "Desarrollador"] }
```

Los roles del 169 son los mismos que tiene `Pedidos` (ID 19). Los padres `1` (Mi Tienda) y `7` (Administracion) existen en producción.

!!! warning "Discrepancia: nombres de campo"
    La primera versión de esta página listaba `Estado` (mayúscula) y `Valor`. Los documentos observados usan **`estado`** en minúscula y **no traen `Valor`** (observado en PreProd y en el módulo `Pedidos` de producción). Esta página sigue lo observado.

!!! tip "Quitar un módulo del menú sin borrarlo"
    Poner `estado` distinto de `"ACTIVO"` lo oculta (reversión de la liberación de métricas).

## Cómo pedírselo a una IA

Prompt: **"Créame este módulo en la DB, dame el JSON"**. La IA debe:

1. Preguntar / leer el `ID` libre y el `ID` del grupo padre (no inventarlos).
2. Definir los `roles` (copiar de un módulo hermano o `["Todos"]`).
3. Elegir el icono CoreUI.
4. Entregar el JSON con los 7 campos y `estado: "ACTIVO"`.
5. Recordar crear el componente Angular (`src/app/views/.../<slug>/`) y
   registrar su ruta en el `routes.ts` del grupo.
6. **No insertar nada**: el JSON lo inserta Iker en Mongo.
