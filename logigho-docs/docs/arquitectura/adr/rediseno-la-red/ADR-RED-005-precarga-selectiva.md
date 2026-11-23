---

## Autor: Iker Acevedo (decisión) · redactado con Claude
Fecha creacion: 2026-11-23
Estado: aceptada (implementada y verificada en local; coste en teléfono lento abierto)

# ADR-RED-005 — Precarga selectiva de módulos en lugar de `PreloadAllModules`

**Autor:** Iker Acevedo (decisión) · redactado con Claude  
**Fecha:** 2026-11-23  
**Estado:** Aceptada por el equipo de construcción (D42, D57); el coste en teléfono lento queda como decisión de Iker.

> Marcas: **observado** (ruta o fuente citada), **Iker** (fecha) o **hipótesis**.

---

## Contexto

El front carga cada módulo por demanda (21 `loadChildren` de área y unos 170 `loadComponent`, según el diseño). Sin precarga, la primera visita a un área espera la descarga de la tabla de rutas y luego la del componente: en un teléfono lento es la mayor parte de la sensación de «congelado» al cambiar de módulo. Antes del rediseño no había `withPreloading` (rige `NoPreloading`).

---

## Opciones consideradas

### Opción A — Sin precarga (la situación de `main`)

**Pros:** cero bytes extra.  
**Contras:** cada primera visita espera la red.

### Opción B — `PreloadAllModules`

**Pros:** sin espera tras el arranque.  
**Contras:** ≈ 4,18 MB comprimidos en 286 archivos (cifra del diseño, sobre un build local); a 1,6 Mb/s son unos 21 s de red compitiendo con los datos de la vista (**cálculo del diseño**). Muchos usuarios nunca ven la mayoría de los módulos: el menú se filtra por rol.

### Opción C — Precarga selectiva por intención y en reposo (elegida)

`PrecargaLaRed`, registrada con `withPreloading(PrecargaLaRed)`:

- **Por intención:** al pasar el puntero 65 ms o tocar un enlace del menú, descarga solo el área y la vista de ese enlace.
- **En reposo:** a los 8 s del primer `NavigationEnd` y con el navegador libre, solo las tablas de rutas de las áreas del menú del usuario (≈ 11 KB comprimidos).
- **Corte por red:** con ahorro de datos o red 2g/3g, solo al tocar.

**Pros:** adelanta la descarga justo cuando hay intención; no hace ninguna petición a la API; las guardas se siguen evaluando al navegar.  
**Contras:** código nuevo que corre en cada navegación.

---

## Decisión

**Se eligió:** Opción C.

**Razón:** cubre la espera de red sin cargar lo que la persona no va a usar. Medido (5 repeticiones intercaladas): con el puntero 200 ms sobre el enlace, la vista llega antes en 3 de 6 casos (81–97 ms frente a 96–175 ms); en el resto, igual.

**Ajuste posterior (D57):** el reposo salía ≈ 2,5 s tras el arranque y descargaba chunks a la vez que los datos de la primera vista, retrasándola 0,3–0,4 s en red lenta. Se movió a 8 s. La carga inicial sigue +1,0–1,3 % en teléfono lento: es el peso del código añadido, no la precarga (retrasarla no lo movió).

---

## Consecuencias

**Positivas:** respuesta más rápida al navegar; 0 peticiones a la API nuevas (paridad 0 diferencias).  
**Negativas:**

- **+1 % de carga inicial** (+0,3 s) y **+33 a +82 ms** al volver a un módulo ya visitado en un teléfono lento simulado. **Decisión abierta de Iker:** aceptarlo a cambio de respuesta inmediata, o quitar la precarga o el indicador en teléfono.
- La medición es de Chromium emulado, no de un teléfono real; con tablas reales los costes serán mayores en los dos builds.
- Una ruta con `canLoad` no se precarga.

---

## Impacto en el código

| Módulo / Repo | Cambio |
|---|---|
| `SitioLogiGho` | `layout/default-layout/precarga/precarga-la-red.ts`, `app.config.ts` (`withPreloading(PrecargaLaRed)`), `default-layout.component.ts` (`precarga.registrarMenu(...)`) |

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-11-23 | Iker Acevedo · redactado con Claude | Primera versión. |

---

## Referencias

- [Movimiento y navegación entre módulos](../../../frontend/movimiento-y-navegacion-la-red.md)
