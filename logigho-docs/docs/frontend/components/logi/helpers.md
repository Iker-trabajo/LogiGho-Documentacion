---
autor: Yadilton Torres
fecha_creacion: 2026-11-24
ultima_actualizacion: 2026-11-24
estado: desarrollo
nivel: 2
---

# Logi: helpers

**Autor:** Yadilton Torres

**Ubicación:** `SitioLogiGho/src/app/components/logi-assistant/helpers/`

Lógica pura, sin Angular: fácil de probar.

---

## `logi-errores.rules.ts`

`explicarAlerta(tipo, titulo, detalle, url?)` convierte una alerta en `{ texto, cara, categoria }`.

Orden en que decide:

1. Errores propios de la pantalla (`errores` en `logi-conocimiento.ts`).
2. Casos frecuentes del proyecto (`CASOS`): guía no encontrada, guía de otra tienda, solo números, archivo o encabezados incorrectos, cantidades, saldo, proveedores, código, correo, contraseña…
3. Categorías generales por palabras: **permiso** («Tu usuario no puede hacer esto…»), **sistema** («No fue culpa tuya…»), **dato** («Hay un dato que corregir…»).
4. Si no reconoce nada: «Uy:» (error) u «Ojo:» (advertencia) más el mensaje.

Los títulos genéricos («Error», «Advertencia», «Atención»…) se ignoran si hay texto. `unirFrases()` une título y detalle sin puntos dobles.

---

## `logi-soporte.rules.ts`

| Función | Qué hace |
|---|---|
| `recortarTecnico(texto)` | Limpia y corta a 300 caracteres |
| `rutaDeServicio(url)` | Ruta del servicio **sin dominio ni parámetros** (pueden llevar correos o teléfonos) |
| `esTecnico(texto)` / `sinTecnicismos(texto)` | Detecta y quita frases técnicas para que al usuario nunca le llegue `Cannot read properties…` |
| `diagnosticar(tecnico, textoUsuario)` | **Causa probable** y **qué revisar** para el equipo: undefined/null (con la propiedad), «is not a function», versión nueva (chunk), errores NG0, sin internet, sesión, formulario y cada código HTTP (0, 401, 403, 404, 400/422, 409, 413, 429, 5xx) |
| `nombreDeRuta(ruta)` | Nombre legible sin ids: `/app/muro-compartido` → «Muro compartido» |
| `navegadorCorto()` | «Chrome en Windows», para el reporte |

---

## `logi-campos.rules.ts`

Ayuda en campos mal escritos. Solo lee el campo: no cambia su valor ni bloquea el formulario, y nunca repite lo que el usuario escribió.

- `revisarCampo(campo)`: al salir de un campo con valor, consejo si está mal (o `null`).
- `formatoDeCampoInvalido(campo)`: al entrar a un campo marcado en rojo, explica el formato.

Reconoce el campo por `formcontrolname`, `name`, `id`, `placeholder`, `aria-label` y su etiqueta. Tipos: correo, teléfono (10 dígitos, sin 57), número de cuenta, documento/NIT, dinero, dirección, guía, medida, peso y cantidad. **Nunca** revisa contraseñas, códigos, tokens, fechas, búsquedas ni textos libres.

---

## `logi-recorrido.ts`

«Muéstrame cómo»: `armarRecorrido(ruta)` arma los pasos con los botones que de verdad están en pantalla y `RecorridoLogi` los recorre señalando cada uno. El componente no lo arma en pantallas sin permiso.

---

## `logi-formas.ts`

Colores (`LOGI_COLORES`), posiciones y expresiones (`ExpresionLogi`) del dibujo del dron.

---

## Pruebas

- `logi-errores.rules.spec.ts`: caso conocido, permisos, sistema, dato, mensaje desconocido y `unirFrases`.
- `logi-soporte.rules.spec.ts`: recorte, ruta sin parámetros, sin tecnicismos, diagnóstico por undefined y por HTTP, nombre de ruta.
- `logi-campos.rules.spec.ts`: teléfono con 57 y con pocos dígitos, correo, documento con puntos, campos que nunca revisa y formato solo en campos en rojo.
