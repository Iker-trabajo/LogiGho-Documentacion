# ADR-002 — Catálogo de experiencias en base de datos, CRUD solo por la Lambda y Native AOT

**Autor:** Iker Acevedo  
**Fecha:** 2026-11-10  
**Estado:** Propuesta (pasa a Aceptada cuando Iker apruebe la salida a producción)

---

## Contexto

El [ADR-001](ADR-001-identidad-y-catalogo-del-servidor.md) dejó el catálogo de experiencias en variables de entorno de Terraform y descartó una colección
administrable por considerarla excesiva. Iker decidió (2026-11-10) que el equipo de desarrollo debe poder **crear, configurar, activar, desactivar y
retirar experiencias desde un panel**, sin tocar la base de datos ni Terraform, y que la Lambda debe **responder lo más rápido posible**. Eso cambia tres cosas:
dónde vive el catálogo, quién puede escribirlo y cómo se compila la Lambda.

---

## Opciones consideradas

### Opción A — Mantener el catálogo en variables de entorno de Terraform
**Pros:** cambios revisados con `plan`, sin escritura de administración que proteger.  
**Contras:** cada cambio exige `terraform apply` de alguien con permisos; no sirve para el panel que se pidió.

### Opción B — Colección `Experiencias` administrada por la API de la misma Lambda (elegida)
**Pros:** el panel actúa sobre la API; las reglas de negocio (clasica intocable, versión, formato de ids) viven en un solo lugar; sin Terraform ni consola.  
**Contras:** una escritura de administración más que proteger; el catálogo ya no está en git; las colecciones se podrían escribir por otras vías (ver el riesgo).

### Opción C — Una Lambda aparte para la administración
**Pros:** aísla la ruta caliente de los usuarios.  
**Contras:** más infraestructura (rol, pipeline, alarmas) para una herramienta de desarrollo. Rechazada: es el mismo dominio y el mismo modelo de datos.

---

## Decisión

1. **Catálogo en la colección `Experiencias`** (un documento por experiencia: `id`, `nombre`, `descripcion`, `estado`, `rolesAcceso`, `temas`, `temaPorDefecto`, `orden`, `version` y auditoría). Se eliminan las variables `EXPERIENCIAS` y `TEMAS`.
2. **CRUD en la misma Lambda** (`/experiencias`), con **lectura abierta** a cualquier rol autenticado (filtrada por la regla de disponibilidad) y **escritura solo** para los roles de `ROLES_ADMIN_EXPERIENCIAS` (variable de entorno; vacía = nadie). Los roles salen de `Users`, nunca del cuerpo.
3. **Reglas de negocio** definidas por Iker: `clasica` intocable (siempre existe, siempre `general`, no se borra ni cambia de estado); **sin reglas de transición** de estado; sin límite de experiencias; ids únicos e inmutables; control de versión; borrado solo en `borrador`/`retirada` y sin usuarios que la tengan guardada; los datos de los usuarios afectados no se tocan (reciben `clasica`).
4. **Auditoría en los logs** de la Lambda (`[AUDIT]`) y en los campos `creadoPor/actualizadoPor` de cada documento. No se crea una colección de historial por ahora; se crea si los controles lo piden.
5. **Rendimiento:** catálogo cacheado 60 s en memoria (una escritura invalida la caché local) y **Native AOT real** (`provided.al2023`, ejecutable `bootstrap`, JSON generado en compilación, solo `BsonDocument`), compilado con Docker sobre Amazon Linux 2023. Meta a medir: arranque en frío < 500 ms y `GET` p95 < 100 ms.
6. **`/metodoGenerico` no se toca:** no se modifica esa Lambda compartida. Se documenta que `Experiencias` y `PreferenciasUsuario` **no** deben escribirse por esa vía; la Lambda ignora documentos inválidos y fuerza `clasica` como defensa.

---

## Consecuencias

**Positivas**
- Cambiar el estado de una experiencia es una llamada a la API, reversible y auditada.
- Un documento manipulado nunca llega a los usuarios ni puede quitar `clasica`.
- Menos tiempo de arranque y de respuesta.

**Negativas / riesgos aceptados**
- Hasta que la Lambda genérica tenga lista blanca de colecciones, nada impide técnicamente escribir estas colecciones por `/metodoGenerico`: lo impide la regla del equipo (backlog: lista blanca).
- El cambio de una experiencia tarda hasta 60 s en verse en todas las instancias.
- Sin confirmación extra al pasar una experiencia a `general` (herramienta solo para desarrollo; reversible cambiando el estado).
- Esta es la primera Lambda con Native AOT real de la plataforma: las demás "AOT" corren en `dotnet8` gestionado. El respaldo es republicar el mismo código como `dotnet10` gestionado (cambia solo el empaquetado y el runtime en Terraform).

---

## Relacionado
[API de administración](../../../api/experiencias.md) · [API de Preferencias](../../../api/preferencias.md) · [Ficha de la Lambda](index.md) · [ADR-001](ADR-001-identidad-y-catalogo-del-servidor.md)

---

## Autor: Iker Acevedo
Fecha creacion: 2026-11-10  
Estado: propuesta
