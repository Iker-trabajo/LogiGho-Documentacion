---

## Autor: Adalberto Gonzalez

Fecha creacion: 2026-05-16  
Estado: desarrollo  
Tipo: vista

# Vista: GestionDevoluciones

**Selector:** `app-gestion-devoluciones`  
**Ubicación:** `src/app/views/logistica/gestion-devoluciones`  
**Acceso:** Autenticado | Rol: ver `ROLES_TRANSPORTADORAS` en el `.component.ts`

---

## ¿Qué hace?

Permite al usuario con rol de logística consultar e importar los registros de devoluciones por transportadora. Presenta dos vistas internas sin cambio de URL: un selector de transportadoras (Vista 1) y la tabla de registros con filtros (Vista 2). Cada transportadora tiene su propia lógica encapsulada en una estrategia (`DevolucionStrategy`).

---

## Ruta

| Ruta                           | Guard      | Parámetros de URL |
| ------------------------------ | ---------- | ----------------- |
| `/app/logistica/devoluciones`  | `AuthGuard`| —                 |

---

## Propiedades clave

| Propiedad                  | Tipo                        | Descripción                                                                 |
| -------------------------- | --------------------------- | --------------------------------------------------------------------------- |
| `strategyActual`           | `DevolucionStrategy \| null`| `null` = Vista 1; con valor = Vista 2. Controla cuál vista renderiza el template. |
| `transportadorasPermitidas`| `string[] \| '*'`           | Calculada en `cargarRol()`. `'*'` habilita todas; array limita por clave de transportadora. |
| `camposFechaOpciones`      | `{ key, label }[]`          | Construida desde `strategy.camposFecha` al seleccionar transportadora. Si está vacía, el filtro de fechas no se muestra. |
| `rowsMemory`               | `TablaRow[]`                | Copia sin filtrar. Se usa para restaurar `rows` cuando el usuario borra los filtros. |

---

## Servicios y endpoints

| Servicio                  | Método               | Endpoint                                     | Cuándo                        |
| ------------------------- | -------------------- | -------------------------------------------- | ----------------------------- |
| `ConsumoGenericoService`  | `consultarGenerico`  | `GET metodoGenerico?coleccion=<coleccion>&mcomp=2` | Al seleccionar transportadora |
| `DecompressionService`    | `decompressZstd`     | —                                            | Al recibir respuesta del backend |

---

## Flujo principal

```
ngOnInit()
  -> cargarRol()          // Lee sessionStorage 'roles_asignados', calcula transportadorasPermitidas

seleccionarTransportadora(key)
  -> DevolucionFactory.crear(key)   // Instancia la estrategia correcta
  -> history.pushState(...)         // Habilita el botón atrás del navegador
  -> cargarTabla()
       -> consultarGenerico(coleccion)
       -> decompressZstd(Resultado)
       -> strategy.sanitize(row)    // Normaliza cada registro
       -> generarColumnas()         // Usa strategy.columnLabels como headers

filtrar()
  -> filtra rowsMemory por searchValue + rango de fechas (campoFecha) + campo desde strategy.camposFecha

volver() -> history.back()
popstate -> resetVista1()           // El botón atrás del navegador vuelve a Vista 1
```

---

## Historial de cambios

| Fecha      | Autor               | Cambio                                                                 |
| ---------- | ------------------- | ---------------------------------------------------------------------- |
| 2026-05-16 | Adalberto Gonzalez  | Creación del módulo con patrón Strategy/Factory                        |
| 2026-05-16 | Adalberto Gonzalez  | Tabla personalizada con paginación, filtro de texto y filtro de fechas |
| 2026-05-16 | Adalberto Gonzalez  | Control de acceso por roles desde `sessionStorage`                     |
| 2026-05-16 | Adalberto Gonzalez  | Navegación interna con `history.pushState` / `popstate`                |

---

## Observaciones

- Las transportadoras con `activa: false` en el array `transportadoras` **siempre** están deshabilitadas sin importar el rol. El rol solo controla las que tienen `activa: true`.
- Para agregar una nueva transportadora: (1) crear su estrategia en `strategies/transportadoras/`, (2) registrarla en `DevolucionFactory`, (3) añadir `{ key, nombre, activa: true }` al array, (4) si tiene permisos especiales agregarla a `ROLES_TRANSPORTADORAS`.
- El filtro de fechas solo aparece si la estrategia define `camposFecha`. Transportadoras nuevas sin ese campo no mostrarán el filtro — esto es intencional para evitar errores en comparaciones de fechas con campos inexistentes.
- La navegación con `popstate` funciona porque el componente nunca cambia de URL: empuja una entrada al historial al entrar a Vista 2 y la consume al volver.
