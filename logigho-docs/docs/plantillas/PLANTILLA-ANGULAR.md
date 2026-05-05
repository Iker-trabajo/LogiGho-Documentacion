---

## Autor:

Fecha creacion: YYYY-MM-DD
Estado: desarrollo | produccion | deprecado
Tipo: vista | componente

# [Vista / Componente]: NombreComponente

**Selector:** `app-nombre`  
**Ubicación:** `src/app/[views|components]/ruta/nombre`  
**Acceso:** Público | Autenticado | Rol: `nombre-rol`

---

## ¿Qué hace?

Qué muestra, para quién, qué problema resuelve al usuario.

---

## Ruta *(solo vistas — eliminar si es componente)*


| Ruta          | Guard       | Parámetros de URL |
| ------------- | ----------- | ----------------- |
| `/app/nombre` | `AuthGuard` | —                 |


---

## Decoradores *(eliminar si no tiene)*


| Decorador | Nombre   | Tipo                 | Descripción     |
| --------- | -------- | -------------------- | --------------- |
| `@Input`  | `nombre` | `tipo`               | Para qué sirve  |
| `@Output` | `evento` | `EventEmitter<tipo>` | Cuándo se emite |


---

## Propiedades clave

> Solo las que no son obvias por su nombre. Si tiene más de 10, documenta solo las críticas.


| Propiedad   | Tipo   | Descripción    |
| ----------- | ------ | -------------- |
| `propiedad` | `tipo` | Para qué sirve |


---

## Servicios y endpoints


| Servicio        | Método      | Endpoint              | Cuándo         |
| --------------- | ----------- | --------------------- | -------------- |
| `NombreService` | `obtener()` | `GET /api/v1/recurso` | Al inicializar |


---

## Flujo principal

Descripccion y flujo principal 

```
ngOnInit()
  -> servicio.obtener()
  -> procesar resultado
  -> renderizar
```

---

## Historial de cambios


| Fecha      | Autor  | Cambio                 |
| ---------- | ------ | ---------------------- |
| YYYY-MM-DD | Nombre | Descripción del cambio |


---

## Observaciones

> Deuda técnica, comportamientos no obvios, decisiones de diseño que no se ven en el código.

- Observación

