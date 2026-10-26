---
autor: Iker Acevedo
fecha_creacion: 2026-10-26
ultima_actualizacion: 2026-10-26
estado: produccion
---

# Novedades Inter (MotivoNovedadInter)

El proveedor va a empezar a enviar las novedades de un pedido como un **arreglo de objetos** en vez de un string plano. Esta página documenta el diseño, cómo activarlo y por qué **no requiere cambio de código** en la lambda.

---

## Antes y después

### Hoy (string plano)

```json
"MotivoNovedadInter": "OTROS / RESIDENTE AUSENTE (Agencia) - https://www.google.com/maps?q=..."
```

### Nuevo (arreglo de objetos)

```json
"MotivoNovedadInter": [
  { "fecha": "2026-10-20T11:10:12-05:00", "estado": "Devuelto al Remitente",     "motivo": "OTROS / NO RECLAMADO EN OFICINA" },
  { "fecha": "2026-10-17T10:03:59-05:00", "estado": "En Proceso de Devolución",  "motivo": "OTROS / RESIDENTE AUSENTE (Agencia)" },
  { "fecha": "2026-10-14T16:06:56-05:00", "estado": "En Proceso de Devolución",  "motivo": "OTROS / RESIDENTE AUSENTE (Agencia)" }
]
```

Cada entrada tiene `fecha`, `estado` y `motivo`. El arreglo guarda el historial completo de novedades del pedido.

---

## Cómo activarlo

Agregar un documento en la colección `ConfiguracionCamposApi`:

```json
{ "Campo": "MotivoNovedadInter", "Activo": true }
```

Con eso, cuando el proveedor empiece a mandar el arreglo, la API lo mostrará en `Resultados`. **No hay que tocar nada más por campo** — la proyección dinámica ya soporta arreglos de objetos.

---

## ¿Requiere cambio de código? No.

Se evaluó cambiar la serialización (de `BsonTypeMapper.MapToDotNetValue` a `doc.ToJson(relaxed)`) para limpiar el "ruido" de MongoDB extended JSON (`$date`, `$numberLong`). La prueba contra un documento real mostró:

- **El arreglo `MotivoNovedadInter` sale limpio** con el serializador actual — no hacía falta ningún cambio.
- El `$numberLong` ya salía como número normal.
- El `$date` persiste **aun en modo relaxed**, porque JSON no tiene tipo fecha nativo; `BsonDateTime` siempre se serializa con `$date`.

Conclusión: el cambio de serialización no aportaba lo que prometía (el arreglo ya funcionaba y el `$date` seguía igual), así que **se revirtió**. El código quedó con `BsonTypeMapper`, igual que antes.

!!! tip "Fechas del arreglo como texto"
    Guardar las fechas del arreglo (`"fecha": "2026-10-20T..."`) como **texto ISO**, no como `Date` de Mongo. Así salen limpias sin tocar el código. Si se guardan como `Date`, aparecerían envueltas en `{ "$date": ... }`.

---

## Ruido de MongoDB extended JSON (contexto)

Campos guardados como tipos especiales de Mongo salen con envoltorio en la respuesta de la API:

```json
"Numeropreenvio": { "$numberLong": "240063080213" }
"InventarioCargaFecha": { "$date": "2026-10-22T22:29:31.018Z" }
```

Esto es un tema transversal de toda la API (no solo de este campo). Limpiarlo de raíz implicaría aplanar manualmente los `BsonDateTime` a string ISO antes de serializar, lo que cambia el formato de salida de campos que ya consumen integradores actuales — es una decisión aparte, no bloqueante para el arreglo de novedades.
