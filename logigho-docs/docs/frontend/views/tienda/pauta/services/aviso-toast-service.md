---

## Autor: Yadilton Torres
Fecha creacion: 2026-10-14
Estado: desarrollo

# Servicio: AvisoToastService

**Ubicación:** `src/app/views/tienda/pauta/services/aviso-toast.service.ts`  
**Scope:** `providedIn: 'root'`

---

## ¿Qué hace?

Muestra un aviso pequeño arriba a la derecha (tarjeta blanca con una barra de color a la izquierda) para explicar algo **sin interrumpir** lo que el usuario está escribiendo. Usa `SweetAlert2` en modo `toast`.

Lo usan la directiva [`appCampoSoloNumeros`](../directives/campo-solo-numeros-directive.md) ("No uses puntos ni comas") y el modal [Importacion Pauta](../../../../components/importacion-pauta.md) ("Solo escribe el Idtienda").

---

## Métodos

### `mostrar(titulo: string, texto: string, tipo: TipoAviso = 'advertencia'): void`

Muestra el aviso durante 4 segundos con barra de tiempo. El tiempo se pausa mientras el mouse está encima.

| Parámetro | Tipo | Descripción |
| --------- | ---- | ----------- |
| `titulo` | `string` | Texto en negrita (ej. "No uses espacios") |
| `texto` | `string` | Explicación corta (ej. "Escribe 1500000, no 1.500.000") |
| `tipo` | `'advertencia' \| 'info'` | `advertencia` = barra naranja, `info` = barra azul |

El título y el texto se escapan antes de armar el HTML, así que se pueden pasar valores escritos por el usuario.

---

## Estilos

Los estilos son **globales** en `src/scss/_aviso-toast.scss` (importado en `src/scss/styles.scss`), porque SweetAlert pinta el aviso fuera de los componentes y los estilos de un componente no lo alcanzan.

| Clase | Qué es |
|---|---|
| `.aviso-toast` | La tarjeta (fondo blanco, borde izquierdo de 4 px, sombra) |
| `.aviso-toast--advertencia` / `.aviso-toast--info` | Color de la barra y del icono (`--logigho-naranja` / `--logigho-azul`) |
| `.aviso-toast__icono`, `__titulo`, `__texto` | Partes del contenido |
| `.aviso-toast__tiempo` | Barra de tiempo, del mismo color |
| `.aviso-toast--entrar` / `--salir` | Entra deslizándose desde la derecha y sale desvaneciéndose (respeta "reducir movimiento") |

---

## Endpoints que consume

Ninguno.

---

## Historial de cambios

| Fecha | Autor | Cambio |
|---|---|---|
| 2026-10-14 | Yadilton Torres | Servicio inicial con estilos globales en `_aviso-toast.scss`. |

---

## Observaciones

- Vive en la carpeta de Pauta porque por ahora solo lo usa este módulo, pero no depende de nada de Pauta: si otro módulo lo necesita, se puede mover a una carpeta compartida.
